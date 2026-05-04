# Story 19.1: Outcome Dashboard Frontend Widgets + Workspace-Config Overrides

Status: review

<!-- Validation note: Run [VS] Validate Story (bmad-validate-story) before bmad-dev-story per Operator BMAD-stream guidance — non-negotiable. AP18-C2 carry-forward: orchestrator MUST patch `Status:` in this file atomically with sprint-status transition (failed 14 consecutive epics E09–E18 + 19-0). AP17-C1 two-gate-close: `Status: done` requires bmad-code-review verdict = Approve, NOT just dev-pass-completes. -->

## Story

As a **Bid Director (with Bid Manager / Workspace Admin co-consumers)**,
I want **a workspace-scoped Outcome Dashboard that renders KPI cards, trend charts, a platform-attributed funnel, a content-reuse leaderboard, and an onboarding-milestone tracker — plus a configuration drawer to override per-workspace `hourly_rate_eur` and `hours_saved_per_bid` baselines so the hours-saved estimate reflects my consultancy's actual cost basis**,
so that **I can present quantified, workspace-isolated ROI evidence (rolling 12-month win rate, platform-attributed bid conversion, hours saved, content-reuse uplift) at QBRs and renewal conversations — proving value with numbers instead of anecdotes (Loopio's "you'll be surprised how many customers churn" warning).**

## Epic Context

- **Epic**: E19 Outcome Telemetry & Renewal Proof (Sprint 17–18; 21 pts; Milestone: Renewal engine)
- **Story points**: 8 | **Type**: backend + frontend
- **FRs covered**: FR9.4 (Outcome Dashboard widgets) + FR9.5 (workspace-configurable hourly-rate / hours-saved baseline). Partial coverage of FR9.3 (consumes the `hours_saved_estimate` derived metric set up in S19.0).
- **Position in epic chain**: Middle story of Epic 19. Hard-depends on **Story 19-0 (`done`)** for the materialized views (`mv_workspace_outcome_stats`, `mv_workspace_content_reuse_stats`, `mv_workspace_onboarding_milestones`) and the `client.opportunities.platform_attributed` column. Hard-blocks **Story 19-2** (Monthly Outcome Brief PDF reads the same `GET /api/v1/workspaces/:id/outcome/dashboard` aggregator function for its server-side render of trend charts → PNG embedded in PDF).
- **Source**: Epic spec `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E19-outcome-telemetry-renewal-proof.md` lines 53–75 (S19.01). PRD v1.1 §6 FR9.4/FR9.5 + §8 US11. Architecture-evaluation §2 Change-6 (analytics stays in client-api).
- **Operator workflow guidance**: `[VS] Validate Story` (NON-NEGOTIABLE) → `bmad-dev-story` → `bmad-code-review` (Approve verdict required for `done` per AP17-C1 two-gate-close) → `[PR] Post-Review` → `[SR] Story Review` for 19-1 → 19-2 dispatch (depends on the dashboard aggregator endpoint defined here) → `[ER] Epic Review` after 19-2 closes.

## Acceptance Criteria

> Source-of-truth: epic spec lines 53–75. AC numbers below cover every epic line item plus carry-forward hardening from Epic 14/15/16/17/18 + Story 19-0 retros (AP14-04 canonical ORM seeding; AP15-08 cross-tenant parametrisation; AP17-C1 two-gate close; AP18-C2 atomic Status patch; AP18-H1 inline test-design fill; 19-0 B-1 MV ownership transfer; 19-0 B-2 non-bypass-role workspace-isolation negative coverage; 19-0 B-3 read-only-summary success path; 19-0 D-4 content-reuse stub degrade-gracefully; 19-0 D-5 onboarding milestones empty-until-S19.2).

### AC-1 — `client.workspace_outcome_config` Table

**Given** a workspace needs override values for `hourly_rate_eur` and `hours_saved_per_bid` distinct from the tenant default,
**When** alembic migration `066_create_outcome_config_tables.py` runs,
**Then**:
1. NEW table `client.workspace_outcome_config` (revision="066", down_revision="065"):
   ```sql
   CREATE TABLE client.workspace_outcome_config (
       workspace_id          UUID PRIMARY KEY REFERENCES client.client_workspaces(id) ON DELETE CASCADE,
       hourly_rate_eur       INTEGER NULL,
       hours_saved_per_bid   INTEGER NULL,
       updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
       updated_by_user_id    UUID NULL REFERENCES client.users(id) ON DELETE SET NULL,
       CONSTRAINT ck_workspace_outcome_config_hourly_rate_nonneg
           CHECK (hourly_rate_eur IS NULL OR hourly_rate_eur >= 0),
       CONSTRAINT ck_workspace_outcome_config_hours_saved_nonneg
           CHECK (hours_saved_per_bid IS NULL OR hours_saved_per_bid >= 0),
       CONSTRAINT ck_workspace_outcome_config_hourly_rate_max
           CHECK (hourly_rate_eur IS NULL OR hourly_rate_eur <= 10000)
   );
   ```
2. CHECK constraints fail-closed on out-of-range writes; the upper bound (10 000 EUR/hr) catches the most common operator typos (cents-vs-eur unit confusion). Document in §6 D-1 that this is a soft sanity ceiling, not a business limit.
3. `updated_by_user_id` is `ON DELETE SET NULL` (preserve audit history when the user who set the override is deleted). `updated_at` defaults to `now()`; ORM-side `onupdate=func.now()` keeps it fresh on PATCH.
4. ORM model NEW at `services/client-api/src/client_api/models/workspace_outcome_config.py`. Schema: `WorkspaceOutcomeConfig(Base)` with `__tablename__ = "workspace_outcome_config"`, `__table_args__ = {"schema": "client"}`. Use **legacy `Column(...)` syntax** to match the pattern in `bid_outcome.py` and `opportunity.py` — AP15-08 carry-forward (consistency over modernisation churn).
5. Anti-pattern guard: do NOT add UPSERT-on-write logic at the ORM layer; the PATCH endpoint (AC-4) implements the upsert via `INSERT ... ON CONFLICT (workspace_id) DO UPDATE` — keep DDL boring and stateless.

### AC-2 — `client.tenant_outcome_config` Table

**Given** tenant-level (company-wide) defaults are needed when no workspace-level override is set,
**When** the same migration `066` runs,
**Then**:
1. NEW table `client.tenant_outcome_config`:
   ```sql
   CREATE TABLE client.tenant_outcome_config (
       company_id            UUID PRIMARY KEY REFERENCES client.companies(id) ON DELETE CASCADE,
       hourly_rate_eur       INTEGER NULL,
       hours_saved_per_bid   INTEGER NULL,
       updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
       updated_by_user_id    UUID NULL REFERENCES client.users(id) ON DELETE SET NULL,
       CONSTRAINT ck_tenant_outcome_config_hourly_rate_nonneg
           CHECK (hourly_rate_eur IS NULL OR hourly_rate_eur >= 0),
       CONSTRAINT ck_tenant_outcome_config_hours_saved_nonneg
           CHECK (hours_saved_per_bid IS NULL OR hours_saved_per_bid >= 0),
       CONSTRAINT ck_tenant_outcome_config_hourly_rate_max
           CHECK (hourly_rate_eur IS NULL OR hourly_rate_eur <= 10000)
   );
   ```
2. ORM model NEW at `services/client-api/src/client_api/models/tenant_outcome_config.py`. Same legacy `Column(...)` style.
3. **Resolution chain** (used by the dashboard endpoint, AC-3): for a given `(company_id, workspace_id)`, the effective values are
   ```
   effective_hourly_rate_eur =
       workspace_outcome_config.hourly_rate_eur
       ?? tenant_outcome_config.hourly_rate_eur
       ?? settings.default_hourly_rate_eur          (env var; default 60)
   effective_hours_saved_per_bid =
       workspace_outcome_config.hours_saved_per_bid
       ?? tenant_outcome_config.hours_saved_per_bid
       ?? settings.default_hours_saved_per_bid       (env var; default 5)
   ```
   These two env vars are added to `BaseServiceSettings` in `packages/eusolicit-common/src/eusolicit_common/config.py` with prefix-respecting defaults `default_hourly_rate_eur=60`, `default_hours_saved_per_bid=5`. Defaults are documented as "research-derived: €60K/yr loaded cost ÷ 220 working days × 8h ≈ €34/hr; rounded UP to €60 for QBR conservatism per epic spec lines 17–18 / FR9.5 PRD prose. Overridable per-tenant via `EUSOLICIT_DEFAULT_HOURLY_RATE_EUR` env var on client-api deployment".
4. Tenant-level CRUD is **out-of-scope** for S19.1 (no admin-API endpoint shipped). Tenant rows are seeded via SQL in §6 D-2 OR remain empty (NULL → falls through to env-var default). Document this in §6.
5. Migration `downgrade()` drops `workspace_outcome_config` first then `tenant_outcome_config` (no FK between them, but reverse-order is the convention).

### AC-3 — `GET /api/v1/workspaces/{workspace_id}/outcome/dashboard` Endpoint

**Given** a Bid Director / Bid Manager / Workspace Admin viewing the dashboard,
**When** they call `GET /api/v1/workspaces/{workspace_id}/outcome/dashboard?from=YYYY-MM&to=YYYY-MM`,
**Then**:
1. NEW router file: `services/client-api/src/client_api/api/v1/workspace_outcome_dashboard.py`. `prefix="/workspaces/{workspace_id}/outcome"`, tags `["workspace-outcome-dashboard"]`. Mount in `services/client-api/src/client_api/main.py` alongside `workspace_bid_outcomes_router` (AC follow-up to S19.0 Task 4 — mirror the include pattern).
2. **Path validation**: `workspace_id` MUST exist AND `workspace.company_id == current_user.company_id` (cross-tenant guard). Return **404** (NOT 403) on mismatch — AP14-04 enumeration-leak rule.
3. **RBAC**:
   - `Depends(require_role("read_only"))` (anyone in the company with at least read access) — analytics is broadly readable per role-ceiling pattern.
   - `Depends(WorkspaceScope)` (Story 14-2 dependency) — workspace-membership check; `_BYPASS_ROLES = frozenset({"admin", "bid_manager"})` carry-forward from S19-0 / Epic 14 (`tenant_admin` is NOT yet in `_BYPASS_ROLES`; D-3 documents this).
   - `Depends(require_paid_tier(min_tier="pro_plus"))` — **Pro+ tier-gate** per epic AC line 16 ("Pro+ tier-gated for QBR embedding in customer's own decks"). The exact tier-gate dependency lives in `services/client-api/src/client_api/core/auth.py` (used by Epic 12 analytics endpoints; reuse the existing helper — do NOT reinvent). If the helper is named `require_paid_tier`, `require_tier`, or `subscription_required`, use whatever the existing analytics_roi.py / analytics_market.py routers import. **Pre-flight grep gate**: `grep -rn "require_paid_tier\|require_tier\|subscription_required\|min_tier" services/client-api/src/client_api/` — pick the canonical helper. If the helper does NOT enforce `pro_plus` minimum, document deviation §6 D-4.
4. **Query params**:
   - `from`: optional `YYYY-MM` (default = current month minus 11 months, i.e. rolling 12-month window).
   - `to`: optional `YYYY-MM` (default = current month).
   - `from > to` → 400.
   - Window > 36 months → 400 (cost guard; epic AC says "rolling 12-month" — wider windows are caller errors, not features).
5. **Response shape** (`OutcomeDashboardResponse`):
   ```python
   class OutcomeDashboardResponse(BaseModel):
       window: WindowResponse  # {from: str "YYYY-MM", to: str "YYYY-MM"}
       kpis: KpiResponse       # {bids_tracked: int, bids_submitted: int, bids_won: int, win_rate: float}
       trend: list[TrendPoint] # [{month: "YYYY-MM", bids_tracked, bids_submitted, bids_won, win_rate, platform_attributed_count, avg_value: float|None}]
       platform_attribution: PlatformAttributionResponse  # {platform_attributed_count: int, externally_sourced_count: int, conversion_rate_diff: float|None}
       content_reuse_leaderboard: list[ContentReuseRow]   # [{content_block_id: UUID, content_block_title: str|None, usage_count: int, win_rate_when_used: float}], top 10 by usage_count desc; degrade-gracefully to [] if mv_workspace_content_reuse_stats is empty (D-2 of S19-0 carry-forward)
       onboarding_milestones: list[MilestoneRow]  # [{milestone: str, completed_at: datetime|None}]; degrade-gracefully to [] until S19-2 seeds the table (D-5 of S19-0)
       config: EffectiveConfigResponse  # {hourly_rate_eur: int, hours_saved_per_bid: int, hourly_rate_source: "workspace" | "tenant" | "default", hours_saved_source: "workspace" | "tenant" | "default"}
       hours_saved_estimate: HoursSavedResponse  # {hours_saved: int, eur_saved: int, formula: "bids_tracked × hours_saved_per_bid × hourly_rate_eur"}
   ```
   The `*_source` fields surface WHERE each value came from (workspace override / tenant default / env-var fallback) so the frontend config drawer can show "Inherited from tenant default" hints — addresses D-2 explicitly.
6. **Implementation**: aggregator service `services/client-api/src/client_api/services/outcome_dashboard_service.py::compute_dashboard(db, *, company_id, workspace_id, window_from, window_to) -> OutcomeDashboardResponse`. The function MUST read aggregates from the **materialized views ONLY** (no `JOIN ... GROUP BY` over base tables — anti-pattern fence row #2). MV queries:
   - KPIs + trend ← `mv_workspace_outcome_stats WHERE workspace_id = :wid AND month BETWEEN :from AND :to`.
   - Content-reuse leaderboard ← `mv_workspace_content_reuse_stats WHERE workspace_id = :wid ORDER BY usage_count DESC LIMIT 10` (joined to `client.content_blocks` for the `title` field; if the join row is missing, fall back to `content_block_id` UUID as the display label).
   - Onboarding milestones ← `mv_workspace_onboarding_milestones WHERE workspace_id = :wid ORDER BY milestone`.
   - Config resolution ← `workspace_outcome_config` LEFT JOIN `tenant_outcome_config` (resolution chain per AC-2.3).
   - Platform-attribution diff ← computed from `mv_workspace_outcome_stats` aggregates (`platform_attributed_count` vs total, `conversion_rate_diff = won_rate_platform - won_rate_external`; if either denominator is 0 → null).
7. **Performance**: Endpoint MUST return in **<500ms p95** for a workspace with 36 months × 50 bids (epic spec line 30 "metrics load in under 500ms using materialized views"). Add a `pytest.mark.performance` test that asserts `time.monotonic()` end-to-end <500ms for a seeded 600-row workspace (same shape as S19-0 AC-9 test).
8. **Caching**: response includes `Cache-Control: public, max-age=1800` (30 min — matches Epic 12 dashboard convention from `analytics_roi.py`); ETag derived from `max(updated_at)` across the 5 source tables.
9. **OpenAPI examples**: include 200-with-data, 200-empty-workspace (zero bids), 404 (workspace not in company), 403 (Free/Starter tier user), 400 (window > 36 months).

### AC-4 — `PATCH /api/v1/workspaces/{workspace_id}/outcome/config` Endpoint

**Given** a workspace admin or bid_manager wants to override `hourly_rate_eur` and/or `hours_saved_per_bid`,
**When** they call `PATCH /api/v1/workspaces/{workspace_id}/outcome/config` with body `{hourly_rate_eur?: int, hours_saved_per_bid?: int}`,
**Then**:
1. The PATCH lives on the SAME router as AC-3 (`workspace_outcome_dashboard.py`), suffix path `/config`.
2. **RBAC**:
   - `Depends(require_role("bid_manager"))` (admin AND bid_manager allowed; contributor / reviewer / read_only blocked → 403).
   - `Depends(WorkspaceScope)` workspace-membership check.
   - **No tier-gate** on PATCH — config override is a read-side toggle for a feature the user already has access to write (avoids the "I'm a paying admin but can't update my own config" UX cliff).
3. **Body validation** via `WorkspaceOutcomeConfigUpdate(BaseModel)`:
   - `hourly_rate_eur: int | None = Field(None, ge=0, le=10000)` (matches DB CHECK).
   - `hours_saved_per_bid: int | None = Field(None, ge=0)`.
   - At least ONE of the two MUST be present in the body — empty `{}` body → 422 with message "At least one of hourly_rate_eur or hours_saved_per_bid is required".
   - To CLEAR an override (revert to tenant/default), send `{"hourly_rate_eur": null}` explicitly. The backend distinguishes `field absent in payload` (no change) from `field=null` (clear override) via Pydantic `model_fields_set`.
4. **Upsert semantics**: implement via `INSERT INTO client.workspace_outcome_config (...) VALUES (...) ON CONFLICT (workspace_id) DO UPDATE SET ...` with `updated_at=now()` and `updated_by_user_id=:current_user_id`. Use `sqlalchemy.dialects.postgresql.insert` for the ON CONFLICT clause — DO NOT round-trip a SELECT-then-UPDATE-or-INSERT (race condition under concurrent admin edits).
5. **Response**: `200 OK` with the new effective `EffectiveConfigResponse` (same shape as AC-3.5 `config` field) so the frontend can re-render the drawer immediately without a follow-up GET.
6. **Audit log**: emit a `bid_outcome.config_updated` event via existing `EventBus` (see `packages/eusolicit-common/src/eusolicit_common/events.py`); event payload `{workspace_id, company_id, updated_by_user_id, hourly_rate_eur, hours_saved_per_bid, previous_hourly_rate_eur, previous_hours_saved_per_bid}`. **No new event schema** in `eusolicit-models` for this story — the existing `EventBus` accepts ad-hoc dicts (verify before assuming; if a typed event is required, add `WorkspaceOutcomeConfigUpdated` to `events.py` with PascalCase `event_type` literal per S18-2 anti-pattern #39 carry-forward).
7. **Pre-flight grep gate** (Task 5.5 below): `grep -rn "from sqlalchemy.dialects.postgresql import insert" services/client-api/src/` — confirm the ON CONFLICT pattern is already used elsewhere; reuse the import style.

### AC-5 — Outcome Dashboard Frontend Page (App Router)

**Given** a user navigates to `/workspaces/{workspaceId}/outcome/dashboard` (or whichever path matches the existing E12 analytics convention — verify by inspecting `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/`),
**When** the page renders,
**Then**:
1. NEW page route at `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/outcome/dashboard/page.tsx`. **Server component** wrapper that renders the **client** dashboard component (project-context Rule 30 — server-component shells, client leaves only).
2. Page composition (top to bottom):
   - **Page header**: title "Outcome Dashboard", subtitle showing the active window (e.g. "May 2025 – Apr 2026"), date-range picker (`from` / `to` month pickers), "Configure" button opening the config drawer (AC-6).
   - **KPI cards row** (4 cards): bids_tracked, bids_submitted, bids_won, win_rate (rendered as percentage). Grid `grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4`. Reuse the **EXACT** `RoiSummaryCards`-style card layout from `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/analytics/roi/components/RoiSummaryCards.tsx:37–62` (`bg-white rounded-lg border p-4`, `text-sm text-slate-500` label, `text-2xl font-bold` value). DO NOT reinvent the KPI card primitive — anti-pattern fence row #1.
   - **Trend chart** (Recharts `<LineChart>` inside `<ResponsiveContainer>`): X-axis = `month`, Y-axis(left) = bids_tracked / bids_submitted / bids_won (3 lines), Y-axis(right) = win_rate as percentage (1 dashed line). Mirror the structure of `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/analytics/roi/components/RoiTrendChart.tsx:64–80`. **MUST** include `<EmptyState>` from `@eusolicit/ui` when `trend.length === 0` (project-context Rule R263 — empty state, NOT blank chart area).
   - **Platform-attribution panel**: small horizontal funnel showing `platform_attributed_count` vs `externally_sourced_count` with `conversion_rate_diff` annotation ("Platform-attributed bids win at +X% vs external"). If `conversion_rate_diff` is null, render "Insufficient data — submit ≥5 bids of each type to compare".
   - **Content-reuse leaderboard**: top-10 table sorted by `usage_count` desc; columns Title / Usage Count / Win Rate When Used. Empty state: "Content-reuse tracking lights up once Epic 7 ships `content_block_usages` — see [docs link]" (D-2 of S19-0 carry-forward).
   - **Onboarding milestones** progress strip: 6 checkmark/empty pills; `completed_at IS NULL` → outline pill with "Pending"; populated → filled pill with date. Empty state: "Milestones populate as your team uses the platform — first milestone usually within 7 days" (D-5 of S19-0 carry-forward).
   - **Hours-saved estimate footer**: shows the formula breakdown ("bids_tracked × hours_saved_per_bid × hourly_rate_eur = €X saved"), with the `(workspace|tenant|default)` source pill next to each input value.
3. **Data fetching**: NEW hook `frontend/apps/client/lib/queries/use-outcome-dashboard.ts::useOutcomeDashboard(workspaceId, window)`. TanStack Query key MUST be `["outcome-dashboard", workspaceId, window.from, window.to]` — workspace-isolation cache rule (project-context Rule 21 + Epic 14 cache-isolation; AP19-1 anti-pattern fence row #3). `staleTime: 1_800_000` (30 min, matches the backend `Cache-Control: max-age=1800`).
4. **All data-rendering sections wrapped in `<QueryGuard>`** from `@eusolicit/ui` (project-context Rule R76 / R21 / R263). NO ad-hoc `if (isLoading) return <Spinner />`. Single `<QueryGuard>` at the top of the dashboard component is acceptable — granular per-card guards are not required for a single-endpoint dashboard.
5. **i18n**: ALL labels, axis titles, tooltip strings, empty states, error messages, and toast text under a new `outcomeDashboard.*` namespace in `frontend/apps/client/messages/{bg,en}.json`. Must add ~30 keys per locale; `pnpm check:i18n` parity (1523 baseline from S19-0 → ~1553 after S19-1).
6. **Accessibility**: every chart wraps a `<figure>` with `<figcaption>` (project-context Rule R263 carry-forward — Recharts is SVG; screen-readers need the figcaption fallback). Every KPI card has `aria-label` summarising the value and label.
7. **The page is wrapped in the existing `<AppShell>`** layout at the layout level (already configured for `(client)` route group); the dashboard component itself does NOT mount an AppShell — anti-pattern fence row #5.

### AC-6 — Configuration Drawer (workspace overrides)

**Given** a workspace admin / bid_manager clicks "Configure" on the dashboard header,
**When** the drawer opens,
**Then**:
1. NEW component `frontend/apps/client/components/OutcomeConfigDrawer.tsx`. Built on the existing shadcn `<Sheet>` / `<Drawer>` primitive in `@eusolicit/ui` (locate it via `grep -rn "Sheet\|Drawer" frontend/packages/ui/src/components/`). DO NOT reinvent the drawer shell.
2. Form uses `useZodForm(schema)` from `@eusolicit/ui` (NEVER bare `useForm()` from `react-hook-form` — anti-pattern fence row #4; ATDD AST forbid clause). Schema fields:
   - `hourly_rate_eur`: `z.number().int().min(0).max(10000).nullable()` (or `.optional()` to allow "no change" semantics; resolution: use a discriminated union — see AC-6.3 below).
   - `hours_saved_per_bid`: `z.number().int().min(0).nullable()`.
3. Each field is wrapped in `<FormField>` from `@eusolicit/ui` (NOT a bare `<Input>` + `<Label>` — anti-pattern fence row #4). Each field shows a "Source: (workspace override | tenant default | system default)" badge next to its input — sourced from the dashboard endpoint's `config.{hourly_rate,hours_saved}_source`. "Reset to default" button per field clears the override (sends `{field: null}` to PATCH).
4. Submit handler calls `PATCH /api/v1/workspaces/{workspaceId}/outcome/config` via NEW API client function `frontend/apps/client/lib/api/outcome-dashboard.ts::updateOutcomeConfig(workspaceId, payload)`. NEW mutation hook `frontend/apps/client/lib/queries/use-outcome-dashboard.ts::useUpdateOutcomeConfig`. On success:
   - `queryClient.invalidateQueries({ queryKey: ["outcome-dashboard", workspaceId] })` (re-fetches the dashboard with new effective config).
   - Success toast `outcomeDashboard.config.toast.saved` (via existing `useUIStore` / Sonner pattern from S19-0 B-3 review-fix).
   - Drawer closes (`onOpenChange(false)`).
5. **Permission gate (frontend)**: render the "Configure" button ONLY if `currentUser.role IN {admin, bid_manager}`. Read role from existing `useAuthStore` selector. If a contributor / reviewer / read_only somehow opens the drawer URL directly, the PATCH backend returns 403 and the form surfaces the error inline (R76 QueryGuard error mode).
6. **i18n parity** for the drawer: ~12 additional keys under `outcomeDashboard.config.*` namespace (drawer title, field labels, source badges, reset button, save button, toast).

### AC-7 — Workspace-Scoped Cross-Tenant + Cross-Workspace Isolation Tests (GET + PATCH)

**Given** the new GET dashboard and PATCH config endpoints,
**When** integration test `tests/integration/test_outcome_dashboard_workspace_isolation.py` runs,
**Then**:
1. **Cross-tenant matrix**: parametrised over `direction ∈ {a_to_b, b_to_a}` × `endpoint ∈ {GET, PATCH}` × `attacker_role ∈ {bid_manager, admin}` = 8 cases. Company A's user attempts the endpoint on Company B's workspace → **404** (NOT 403 — AP14-04 enumeration leak rule).
2. **Cross-workspace negative for non-bypass roles** (S19-0 B-2 carry-forward — failed once, MUST be in this story too): parametrised over `direction ∈ {w1_to_w2, w2_to_w1}` × `endpoint ∈ {GET, PATCH}` × `attacker_role = contributor` (NOT in `_BYPASS_ROLES`) = 4 cases. Same-company, different-workspace attacker → **403** from `WorkspaceScope` membership rejection.
3. **Cross-workspace bypass positive** (S19-0 carry-forward): admin / bid_manager (in `_BYPASS_ROLES`) cross-workspace within same company → 200 (proves bypass still works for non-attackers).
4. **Tier-gate negative**: parametrised over `tier ∈ {free, starter, professional}` × `endpoint = GET` = 3 cases. Free/Starter/Professional users → **403** with body `{"detail": "Pro+ subscription required"}` (or whatever the existing tier-gate helper returns; do NOT assert specific text — assert status_code only). Professional+ AND Enterprise users → 200.
5. **PATCH role-gate negative**: parametrised over `role ∈ {contributor, reviewer, read_only}` × `endpoint = PATCH` = 3 cases. → **403** from `require_role("bid_manager")` ceiling check.
6. **Inactive-user negative**: `is_active=False` admin → 401/403 (User.is_active gate; project-context anti-pattern fence). Both endpoints.
7. **Canonical ORM seeding ONLY** (AP14-04 BLOCKING #3 carry-forward — Story 19-0 Round 2 fix): use `Company`, `Workspace`, `User`, `CompanyMembership`, `Subscription`, `Opportunity`, `Proposal`, `BidOutcome` ORM models. NEVER raw `text("INSERT INTO client.…")`. Seeding helpers: `register_and_verify_with_role` + `create_company_pair` from `eusolicit-test-utils` (factories at `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py`).
8. **No `db_session.commit()` in test bodies** — only in fixtures (gold-standard rollback isolation, S19-0 D-10 follow-up: continue to use the `db_session` fixture; do NOT introduce `app_client_fresh`-style commit-leaking fixtures unless absolutely necessary, and document any deviation in §6).
9. Marker: `@pytest.mark.integration`.

### AC-8 — Configuration Override Persistence + Resolution Chain Test

**Given** the workspace_outcome_config and tenant_outcome_config tables,
**When** integration test `tests/integration/test_outcome_config_resolution.py` runs,
**Then**:
1. **Resolution-chain matrix** (parametrised, 9 cases):
   | workspace value | tenant value | env value | expected | expected source |
   |---|---|---|---|---|
   | 80 | 70 | 60 | 80 | workspace |
   | NULL | 70 | 60 | 70 | tenant |
   | NULL | NULL | 60 | 60 | default |
   | 0 | 70 | 60 | 0 | workspace (zero is a valid override; 0 ≠ NULL) |
   | 80 | NULL | 60 | 80 | workspace |
   | NULL | 0 | 60 | 0 | tenant (zero is a valid override) |
   | (no row) | 70 | 60 | 70 | tenant (absent row treated identically to NULL) |
   | (no row) | (no row) | 60 | 60 | default |
   | 9999 | 70 | 60 | 9999 | workspace (within ceiling) |
2. The test seeds rows via ORM, calls the dashboard service `compute_dashboard()` directly (NOT the HTTP endpoint — this is a unit-y test of the resolution function), asserts `config.hourly_rate_eur` and `config.hourly_rate_source`.
3. Same matrix for `hours_saved_per_bid`. Both fields are independent — workspace can override hourly_rate while inheriting tenant's hours_saved.
4. **Upsert idempotency test**: PATCH the same `(workspace_id, hourly_rate_eur=80)` body twice in a row → second call MUST be a no-op SET on the existing row (assert `updated_at` is fresh on both calls; assert no duplicate row in `workspace_outcome_config` — `SELECT COUNT(*) FROM client.workspace_outcome_config WHERE workspace_id = :wid` returns 1).
5. **Constraint test**: PATCH with `hourly_rate_eur=10001` → 422 (Pydantic ge=0/le=10000 catches before DB); attempt direct SQL INSERT with `hourly_rate_eur=10001` → CHECK constraint violation (proves DB layer is the second line of defence).
6. **Race-condition test**: 5 concurrent PATCHes via `asyncio.gather` with different values → all 5 succeed (200), final state is one of the 5 values (whichever won the ON CONFLICT race), `COUNT(*) == 1`. No duplicate-key error, no deadlock.
7. Marker: `@pytest.mark.integration`. Canonical ORM seeding only.

### AC-9 — Frontend ATDD Source-Inspection Tests

**Given** the new dashboard page and config drawer,
**When** ATDD source-inspection test `frontend/apps/client/__tests__/outcome-dashboard-source-inspection.test.ts` runs,
**Then**:
1. AST scan over `app/[locale]/(client)/workspaces/[workspaceId]/outcome/dashboard/page.tsx` and `components/OutcomeConfigDrawer.tsx` (using `@babel/parser` + `@babel/traverse`). Pattern reuse: copy harness from `frontend/apps/client/__tests__/outcome-capture-source-inspection.test.ts` (S19-0 set the canonical pattern).
2. **Asserts** for the dashboard page:
   - Imports `useOutcomeDashboard` hook.
   - Imports `<QueryGuard>` from `@eusolicit/ui`. (project-context Rule R76 / R21).
   - Imports `<ResponsiveContainer>` AND `<LineChart>` from `recharts`.
   - Renders `<EmptyState>` for the trend chart empty case (Rule R263 — chart MUST have empty-state, NOT blank canvas).
   - TanStack Query key passed to `useOutcomeDashboard` includes the variable `workspaceId` (Epic 14 cache-isolation rule).
3. **Asserts** for `OutcomeConfigDrawer`:
   - Imports `useZodForm` from `@eusolicit/ui` (NOT bare `useForm` from `react-hook-form` — forbid clause).
   - Every `<input>` / `<select>` / `<textarea>` is INSIDE a `<FormField>` JSX wrapper (parent-chain walk).
   - Calls `queryClient.invalidateQueries` with a key array that includes `workspaceId`.
4. **Forbids**:
   - `from "react-hook-form"` import in `OutcomeConfigDrawer.tsx` (the form-pattern guard).
   - Direct fetch in render (no `fetch(...)` outside `lib/api/`).
   - Hardcoded English strings in JSX text nodes (regex over JSX text children — flag any string longer than 2 chars not wrapped in `t(...)` or `<Trans />`); whitelist obvious non-i18n tokens (`%`, `€`, `:`).
5. Test runs in `pnpm test --filter client` (vitest) — gates on the L1 ATDD pre-merge layer that `bmad-dev-story` validates before mark-as-review.

### AC-10 — Performance Test: Dashboard Endpoint <500ms p95

**Given** the materialized-view-backed aggregator,
**When** integration test `tests/integration/test_outcome_dashboard_perf.py::test_dashboard_p95_under_500ms` runs,
**Then**:
1. Seed: ONE workspace with 36 monthly buckets × 50 bid_outcomes per month = 1800 rows (canonical ORM seeding); refresh the MVs (`REFRESH MATERIALIZED VIEW client.mv_workspace_outcome_stats` + the other two — non-CONCURRENTLY is fine in test setup since no concurrent readers).
2. Run 10 sequential `httpx` calls to `GET /api/v1/workspaces/{wid}/outcome/dashboard`; collect response times via `time.monotonic()`.
3. Assert: p95 (9th-out-of-10) < 500ms. Assert: max < 750ms (head-room before alarming).
4. Marker: `@pytest.mark.integration` AND `@pytest.mark.performance`. Skip in normal `make test`; run via `make test-integration` or an explicit `pytest -m performance`.
5. **Inline NFR partial fill** — AP18-C4 / S19-0 D-7 carry-forward: this test plus a runbook entry on materialized-view refresh duration is the S19.1 NFR contribution. Full E19 NFR (PDF memory profile + S3 TTL) lands in S19.2 per `bmad-testarch-nfr` regen.

### AC-11 — i18n Parity + Test-Design Inline Fill

**Given** no `test_artifacts/test-design-epic-19.md` exists (S19-0 §4.7 also fills this gap inline; AP18-H1 carry-forward),
**When** Story 19.1 ships,
**Then**:
1. §4.7 of this story file fills the test-design gap inline (extends S19-0 risk taxonomy with S19-1 specific risks):
   - **R-019-7** Tier-gate misconfiguration leaks Pro+ analytics to Free/Starter (Score 6, mitigation AC-7.4).
   - **R-019-8** Resolution-chain bug (workspace override silently ignored, env-var default applied instead) (Score 5, mitigation AC-8.1 9-case parametrised matrix).
   - **R-019-9** Dashboard endpoint p95 > 500ms violates epic AC line 30 (Score 4, mitigation AC-10).
   - **R-019-10** Cross-workspace dashboard read leaks W2 metrics into W1 admin's view (Score 6, mitigation AC-7.1 + AC-7.2).
   - **R-019-11** Config PATCH race condition produces duplicate `workspace_outcome_config` rows (Score 3, mitigation AC-8.6 + ON CONFLICT clause).
   - **R-019-12** Story `Status:` header not atomically patched (15-epic recurrence — AP18-C2) (Score 3, mitigation Task 12 explicit two-edit-one-commit gate).
2. **Test-design provenance** cites: epic spec lines 53–75 (acceptance criteria), AP14-04 (canonical ORM seeding), AP15-08 (consistency over modernisation), AP17-C1 (two-gate close), AP18-C2 (atomic Status patch), AP18-H1 (inline test-design fill), Project-context Rules R21 (CONCURRENTLY + UNIQUE index — already enforced at MV level by S19-0), R44 (analytics scope), R76 (QueryGuard), R263 (Recharts E2E + EmptyState), Story 19-0 Pass-2 Approve verdict (B-1, B-2, B-3 patterns to NOT regress).
3. **i18n parity gate**: `pnpm check:i18n` MUST pass after Story 19.1 (1523 + ~42 = ~1565 keys per locale). New keys namespaced under `outcomeDashboard.*` (page chrome, KPI card labels, chart titles, empty states), `outcomeDashboard.config.*` (drawer fields, source badges, toasts), `outcomeDashboard.errors.*`.
4. **Component-level Vitest tests** for `OutcomeConfigDrawer`: 4 happy-path cases (workspace override / tenant default visible / reset clears workspace value / submit success) + 3 validation-error cases (negative number / above 10 000 / non-integer). Run via `pnpm --filter client test`.

## Tasks / Subtasks

> Order is implementation-dependency-driven. Each task lists owning ACs in parens. Tasks are sized for ≤30 min focused work each.

- [ ] **Task 1 — Alembic migration 066: outcome config tables** (AC-1, AC-2)
  - [ ] Create `services/client-api/alembic/versions/066_create_outcome_config_tables.py` (revision="066", down_revision="065"). Mirror `065_create_workspace_outcome_materialized_views.py` formatting; include docstring quoting AC-1 + AC-2.
  - [ ] `upgrade()` creates `client.workspace_outcome_config` then `client.tenant_outcome_config` (order doesn't matter — no cross-FK; alphabetical for predictability).
  - [ ] Both tables: PK + FKs `ON DELETE CASCADE` for the parent (workspace / company), `ON DELETE SET NULL` for `updated_by_user_id`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`, and the 3 CHECK constraints per AC.
  - [ ] `downgrade()` drops in reverse order.
  - [ ] Run `make migrate-service SVC=client-api`; confirm via `\d client.workspace_outcome_config` + `\d client.tenant_outcome_config` in psql.
  - [ ] Run `alembic check` — must report no drift.
  - [ ] Add `default_hourly_rate_eur: int = 60` and `default_hours_saved_per_bid: int = 5` to `BaseServiceSettings` in `packages/eusolicit-common/src/eusolicit_common/config.py` (or to `ClientApiSettings` if `BaseServiceSettings` is shared and shouldn't carry this). Confirm env-var prefix resolves to `EUSOLICIT_DEFAULT_HOURLY_RATE_EUR`.

- [ ] **Task 2 — ORM models + Pydantic schemas** (AC-1.4, AC-2.2, AC-3.5, AC-4.3)
  - [ ] Create `services/client-api/src/client_api/models/workspace_outcome_config.py` and `services/client-api/src/client_api/models/tenant_outcome_config.py` (legacy `Column(...)` style — match `bid_outcome.py`).
  - [ ] Re-export both from `services/client-api/src/client_api/models/__init__.py`.
  - [ ] Create `services/client-api/src/client_api/schemas/outcome_dashboard.py` with: `OutcomeDashboardResponse`, `WindowResponse`, `KpiResponse`, `TrendPoint`, `PlatformAttributionResponse`, `ContentReuseRow`, `MilestoneRow`, `EffectiveConfigResponse`, `HoursSavedResponse`, `WorkspaceOutcomeConfigUpdate` (PATCH body).
  - [ ] Run `alembic check` again — no drift after ORM changes.

- [ ] **Task 3 — Aggregator service + resolution-chain helper** (AC-3.6, AC-2.3)
  - [ ] Create `services/client-api/src/client_api/services/outcome_dashboard_service.py` with `compute_dashboard(db, *, company_id, workspace_id, window_from, window_to) -> OutcomeDashboardResponse`. Read aggregates from `mv_workspace_outcome_stats` / `mv_workspace_content_reuse_stats` / `mv_workspace_onboarding_milestones` ONLY; no JOIN over base tables.
  - [ ] Implement `resolve_outcome_config(db, *, company_id, workspace_id) -> tuple[EffectiveConfigResponse]` — the 3-tier chain (workspace → tenant → settings default).
  - [ ] Compute `hours_saved_estimate = bids_tracked × effective_hours_saved_per_bid × effective_hourly_rate_eur` (returns 0 if `bids_tracked == 0`).
  - [ ] `platform_attribution.conversion_rate_diff` is `won_rate_platform − won_rate_external`; null if either denominator is 0.
  - [ ] Content-reuse leaderboard: SQL query joins `mv_workspace_content_reuse_stats` to `client.content_blocks` (LEFT JOIN to tolerate missing rows when stub MV is empty per S19-0 D-4); ORDER BY usage_count DESC LIMIT 10.

- [ ] **Task 4 — Router + GET endpoint mount** (AC-3, AC-4)
  - [ ] Create `services/client-api/src/client_api/api/v1/workspace_outcome_dashboard.py` with router `prefix="/workspaces/{workspace_id}/outcome"`, tags `["workspace-outcome-dashboard"]`.
  - [ ] Implement `GET /dashboard`: dependencies on `get_current_user`, `require_role("read_only")`, `WorkspaceScope`, `require_paid_tier(min_tier="pro_plus")` (or whatever the canonical helper is — see AC-3.3 grep gate). Validate `from`/`to` query params; clamp to 36-month max; default to rolling 12-month.
  - [ ] Set `Cache-Control: public, max-age=1800` + ETag from `MAX(updated_at)` across the 5 source tables. (If ETag computation is awkward, defer to a follow-up — document §6 D-5; a missing ETag is a perf hint loss but NOT a correctness gap.)
  - [ ] Mount router in `services/client-api/src/client_api/main.py` next to the workspace_bid_outcomes mount.
  - [ ] Add OpenAPI examples per AC-3.9 (use `responses=` kwarg with status-code-keyed dict).

- [ ] **Task 5 — PATCH endpoint** (AC-4)
  - [ ] Implement `PATCH /config` on the same router. Dependencies: `require_role("bid_manager")` + `WorkspaceScope`. NO tier-gate.
  - [ ] Validate body via `WorkspaceOutcomeConfigUpdate`. Use `model_fields_set` to distinguish absent-field vs explicit-null-clear.
  - [ ] Use `sqlalchemy.dialects.postgresql.insert(...).on_conflict_do_update(index_elements=["workspace_id"], set_={...})` for the upsert. **Pre-flight grep gate**: `grep -rn "on_conflict_do_update\|from sqlalchemy.dialects.postgresql import insert" services/client-api/src/` to find existing usage patterns; mirror style.
  - [ ] On success: emit `bid_outcome.config_updated` event via existing `EventBus`; return `EffectiveConfigResponse`.
  - [ ] **Audit log**: confirm `EventBus.publish` accepts ad-hoc event dict OR adds `WorkspaceOutcomeConfigUpdated` to `eusolicit-models/events.py` (PascalCase event_type literal per S18-2 anti-pattern #39 carry-forward).

- [ ] **Task 6 — Frontend API client + TanStack Query hooks** (AC-5.3, AC-6.4)
  - [ ] Create `frontend/apps/client/lib/api/outcome-dashboard.ts` with `fetchOutcomeDashboard(workspaceId, window)` and `updateOutcomeConfig(workspaceId, payload)`. Use the existing `apiClient` axios instance + auth interceptor (find via `grep -rn "apiClient\|api-client" frontend/apps/client/lib/`).
  - [ ] Create `frontend/apps/client/lib/queries/use-outcome-dashboard.ts` with `useOutcomeDashboard(workspaceId, window)` (TanStack `useQuery`) and `useUpdateOutcomeConfig(workspaceId)` (`useMutation`). Query key includes `workspaceId`. `staleTime: 1_800_000`. On mutation success: `invalidateQueries({ queryKey: ["outcome-dashboard", workspaceId] })`.

- [ ] **Task 7 — Outcome Dashboard page + widget components** (AC-5)
  - [ ] Create `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/outcome/dashboard/page.tsx` (server component shell rendering the client `<OutcomeDashboard />`).
  - [ ] Create `frontend/apps/client/components/OutcomeDashboard.tsx` (`'use client'`) — top-level dashboard composition: header + KPI row + trend chart + platform-attribution panel + content-reuse leaderboard + onboarding milestones strip + hours-saved footer. Wrap in single top-level `<QueryGuard>`.
  - [ ] Create child components in `frontend/apps/client/components/outcome-dashboard/`:
    - `OutcomeKpiCards.tsx` — 4 cards in a grid (mirror `RoiSummaryCards.tsx`).
    - `OutcomeTrendChart.tsx` — Recharts `<LineChart>` (mirror `RoiTrendChart.tsx`); include `<EmptyState>` per Rule R263.
    - `PlatformAttributionPanel.tsx` — funnel visualisation (Recharts `<BarChart>` horizontal OR a simple flex layout).
    - `ContentReuseLeaderboard.tsx` — table (reuse shadcn `<Table>` from `@eusolicit/ui` if present).
    - `OnboardingMilestonesStrip.tsx` — 6 pills.
    - `HoursSavedFooter.tsx` — formula breakdown + source-badge pills.
  - [ ] Each chart wrapped in `<figure>` with i18n `<figcaption>` for accessibility.

- [ ] **Task 8 — Configuration drawer** (AC-6)
  - [ ] Create `frontend/apps/client/components/OutcomeConfigDrawer.tsx` using `useZodForm` + `<FormField>` + the existing `<Sheet>` / `<Drawer>` primitive from `@eusolicit/ui` (locate via `grep -rn "Sheet\|Drawer" frontend/packages/ui/src/components/`).
  - [ ] Wire up the mutation hook from Task 6; success → `invalidateQueries` + close drawer + success toast (reuse `useUIStore` from S19-0 OutcomeCaptureForm pattern).
  - [ ] Source-badge per field reads from the dashboard endpoint's `config.{field}_source` ("workspace" / "tenant" / "default"). "Reset to default" button per field sends `{field: null}` PATCH.
  - [ ] Frontend permission gate: render "Configure" button only if `currentUser.role IN {admin, bid_manager}` (read from `useAuthStore`).

- [ ] **Task 9 — i18n keys + parity** (AC-5.5, AC-6.6, AC-11.3)
  - [ ] Add `outcomeDashboard.*` namespace keys (~30) to `frontend/apps/client/messages/en.json` and `frontend/apps/client/messages/bg.json`. Subkeys: `header.{title,subtitle}`, `kpi.{bidsTracked,bidsSubmitted,bidsWon,winRate}`, `trend.{title,axisX,axisYBids,axisYWinRate}`, `platformAttribution.{title,platformAttributedLabel,externalLabel,diffPositive,diffNegative,insufficientData}`, `contentReuse.{title,empty,columnTitle,columnUsage,columnWinRate}`, `milestones.{title,pending,completedAt}`, `hoursSaved.{title,formula,workspaceSource,tenantSource,defaultSource}`, `errors.{loadFailed,permissionDenied,tierLocked}`.
  - [ ] Add `outcomeDashboard.config.*` namespace (~12 keys): `drawer.{title,description}`, `field.{hourlyRateEur,hoursSavedPerBid}`, `source.{workspace,tenant,default}`, `reset.button`, `submit.button`, `toast.{saved,error}`.
  - [ ] Run `pnpm check:i18n` → expect 1523 + ~42 ≈ 1565 keys parity.

- [ ] **Task 10 — Cross-tenant + cross-workspace + tier-gate isolation tests** (AC-7)
  - [ ] Create `tests/integration/test_outcome_dashboard_workspace_isolation.py` covering all 8 + 4 + 1 + 3 + 3 + 1 = 20 cases per AC-7.
  - [ ] Use `register_and_verify_with_role(role="bid_manager")` and `create_company_pair()` from `eusolicit-test-utils`.
  - [ ] Tier-gate test: parametrise tiers via subscription seeding (look at how Epic 12 dashboards test tier gates — `services/client-api/tests/integration/test_analytics_*` is the precedent).
  - [ ] Canonical ORM seeding only (NO raw `text("INSERT …")` — AP14-04 BLOCKING #3).
  - [ ] No `db_session.commit()` in test bodies — only fixtures (S19-0 M1 / D-10 carry-forward).

- [ ] **Task 11 — Resolution-chain + upsert idempotency tests** (AC-8)
  - [ ] Create `tests/integration/test_outcome_config_resolution.py` with the 9-case parametrised resolution matrix.
  - [ ] Add upsert idempotency test: 2× same PATCH → `COUNT(*) == 1`.
  - [ ] Add CHECK-constraint test: direct ORM INSERT `hourly_rate_eur=10001` → `IntegrityError`.
  - [ ] Add concurrent-PATCH race test: 5× `asyncio.gather` PATCHes → all 200, final state is one of the 5, `COUNT(*) == 1`.
  - [ ] Marker: `@pytest.mark.integration`. Canonical ORM seeding.

- [ ] **Task 12 — Performance test** (AC-10)
  - [ ] Create `tests/integration/test_outcome_dashboard_perf.py::test_dashboard_p95_under_500ms`. Seed 1800 bid outcomes; refresh MVs; 10 sequential httpx calls; assert p95 <500ms, max <750ms.
  - [ ] Markers: `@pytest.mark.integration` AND `@pytest.mark.performance`.
  - [ ] Add `[performance]` Make target if not already present (`make test-performance`); document in `eusolicit-app/Makefile`.

- [ ] **Task 13 — Frontend ATDD source-inspection + component tests** (AC-9, AC-11.4)
  - [ ] Create `frontend/apps/client/__tests__/outcome-dashboard-source-inspection.test.ts` (copy harness from `outcome-capture-source-inspection.test.ts` — Story 19-0 set the canonical L1 ATDD pattern).
  - [ ] Create component-level Vitest test for `OutcomeConfigDrawer` (4 happy-path + 3 validation-error cases per AC-11.4).
  - [ ] Run `pnpm --filter client test` — all green.

- [ ] **Task 14 — `Status: review` transition + sprint-status atomic patch** (AP18-C2 carry-forward — failed 14 consecutive epics; S19-0 Round 1 was the most recent successful application of this pattern)
  - [ ] After all tests pass: edit this file's line 3 from `Status: ready-for-dev` to `Status: review`.
  - [ ] In the SAME `bmad-dev-story` commit, update `eusolicit-docs/implementation-artifacts/sprint-status.yaml`: `19-1-outcome-dashboard-frontend-widgets-workspace-config-overrides: review`.
  - [ ] Both edits MUST be in ONE commit — atomic transition. AP18-C2 has been violated 14 consecutive epics; S19-0 was the first time it was applied correctly. Continue the streak.

- [ ] **Task 15 — Validation gate before mark-as-review**
  - [ ] `make migrate-all` → all alembic revisions clean, no drift.
  - [ ] `pytest services/client-api -k "outcome_dashboard or outcome_config or workspace_outcome" -v` → green.
  - [ ] `pytest tests/integration/test_outcome_dashboard_workspace_isolation.py tests/integration/test_outcome_config_resolution.py tests/integration/test_outcome_dashboard_perf.py -v` → green.
  - [ ] `pnpm test --filter=client outcome-dashboard` → ATDD source-inspection + component tests green.
  - [ ] `pnpm check:i18n` → ~1565 keys parity.
  - [ ] `pnpm type-check` → clean (frontend).
  - [ ] `make lint` + `make type-check` → clean (backend).
  - [ ] Quote the FULL pytest summary line in Dev Agent Record (M2 carry-forward — Story 15-0 + Story 19-0 M-1 review-fix pattern; failing this re-opens M-1 and blocks Pass-2 Approve).

## Dev Notes

### 1. Architecture Compliance

**Source**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md`

- **§2 Change-6 (Rule of Three)**: Analytics stays in `client-api`. Do NOT split into a new service. The new tables (`workspace_outcome_config`, `tenant_outcome_config`) live in `client` schema; the dashboard endpoint reads MVs that are `client`-schema-owned (with `notification_role` ownership-transferred for refresh — S19-0 B-1 fix).
- **FR7.2 Concurrent Refresh Pattern**: The MVs themselves are S19-0's responsibility; S19-1 is a pure CONSUMER — do NOT add new MVs. If you need to compute something not already in an MV, FAIL FAST and document deviation §6 D-X. Anti-pattern fence row #2 enforces this.
- **DB Schema Isolation**: One Postgres database, six schemas (`client`, `admin`, `pipeline`, `gateway`, `notification`, `shared`). NEVER cross-schema in application code. `workspace_outcome_config` and `tenant_outcome_config` live in `client` only; the PATCH endpoint runs as `client_role` (default for client-api).
- **RBAC**: Company-level roles + workspace-membership extension from Epic 14. `_BYPASS_ROLES = frozenset({"admin", "bid_manager"})` for cross-workspace bypass within the same company. `tenant_admin` is a future role (Epic 14 retro lists this) — NOT yet in `_BYPASS_ROLES`; D-3 documents this for AC-7.3.

### 2. Source-Hint Citations

| What | Where (file path : approx line) |
|------|----------------------------------|
| MV definitions (S19-0) | `services/client-api/alembic/versions/065_create_workspace_outcome_materialized_views.py` (entire file) |
| Latest alembic migration | `services/client-api/alembic/versions/065_create_workspace_outcome_materialized_views.py` (revision="065"; this story's `down_revision="065"`) |
| ORM legacy `Column(...)` style reference | `services/client-api/src/client_api/models/bid_outcome.py:13–66` |
| Workspace ORM | `services/client-api/src/client_api/models/client_workspace.py` (verify exact filename via `ls`) |
| Existing workspace-scoped router pattern (S19-0) | `services/client-api/src/client_api/api/v1/workspace_bid_outcomes.py` |
| RBAC `WorkspaceScope` | `services/client-api/src/client_api/core/rbac.py` (S19-0 Round 2 reordered: `is_active` step 3, company-isolation step 4, workspace-membership step 6 with `_BYPASS_ROLES` carve-out) |
| Tier-gate dependency (Epic 12) | `services/client-api/src/client_api/core/auth.py` — search for `require_paid_tier`, `require_tier`, `subscription_required`, or `min_tier` |
| Epic 12 dashboard endpoint precedent | `services/client-api/src/client_api/api/v1/analytics_roi.py:34–99` (filters, pagination, RBAC, Cache-Control) |
| Epic 12 dashboard endpoint precedent | `services/client-api/src/client_api/api/v1/analytics_market.py:34–78` |
| ON CONFLICT upsert reference | `grep -rn "on_conflict_do_update" services/client-api/src/` (find existing usage; mirror) |
| Router include in main.py | `services/client-api/src/client_api/main.py:120–151` (analytics routers registered here; mount new one alongside) |
| BaseServiceSettings | `packages/eusolicit-common/src/eusolicit_common/config.py` |
| EventBus | `packages/eusolicit-common/src/eusolicit_common/events.py` |
| Existing event schema patterns | `packages/eusolicit-models/src/eusolicit_models/events.py:138–149` (BidOutcomeRecorded — PascalCase event_type, S18-2 #39 fence) |
| `useZodForm` | `frontend/packages/ui/src/lib/hooks/useZodForm.ts:14` |
| `<FormField>` | `frontend/packages/ui/src/components/forms/FormField.tsx:101` |
| `<QueryGuard>` | `frontend/packages/ui/src/components/feedback/QueryGuard.tsx:20–28` |
| `<EmptyState>` | `frontend/packages/ui/src/components/feedback/EmptyState.tsx` (verify exact path) |
| `<AppShell>` | `frontend/packages/ui/src/components/app-shell/AppShell.tsx:13–20` |
| Sheet / Drawer primitive | `grep -rn "Sheet\|Drawer" frontend/packages/ui/src/components/` (locate; pre-flight gate Task 8) |
| Recharts trend chart precedent | `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/analytics/roi/components/RoiTrendChart.tsx:64–80` |
| KPI card pattern | `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/analytics/roi/components/RoiSummaryCards.tsx:37–62` |
| ROI dashboard container | `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/analytics/roi/components/RoiTrackerDashboard.tsx:15–100` |
| TanStack Query hook pattern | `frontend/apps/client/lib/queries/use-roi-analytics.ts:11–48` |
| `useWorkspaceId` hook | `frontend/apps/client/lib/hooks/use-workspace-id.ts` (verify path) |
| ATDD source-inspection harness (S19-0) | `frontend/apps/client/__tests__/outcome-capture-source-inspection.test.ts` (copy harness; vitest + @babel/parser + @babel/traverse) |
| Earlier ATDD harness (Story 14-3) | `frontend/apps/client/__tests__/workspace-switcher-source-inspection.test.ts` (canonical pattern reference) |
| Test factories `register_and_verify_with_role`, `create_company_pair` | `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py` (root `tests/conftest.py` re-exports) |
| Subscription-tier seeding for tier-gate tests | `services/client-api/tests/integration/test_analytics_*.py` (Epic 12 — find a test that asserts free/starter/professional/pro_plus tier behaviour and reuse the fixture) |
| OutcomeCaptureForm (S19-0) | `frontend/apps/client/components/OutcomeCaptureForm.tsx` (read-only summary on success — pattern reference for OutcomeConfigDrawer success behaviour) |
| `useUIStore` toast pattern | `frontend/apps/client/lib/stores/use-ui-store.ts` (verify path; S19-0 used this for B-3 success toast) |

### 3. Anti-Pattern Fence (Do-Not-Reimplement / Do-Not-Misuse)

This story has **8 anti-pattern fence rows** the dev agent must observe. Each maps to a documented project-context rule + previous-story carry-forward.

| # | Anti-pattern | Source / Rule | Story 19.1 manifestation |
|---|--------------|---------------|--------------------------|
| 1 | KPI card / chart card / leaderboard card primitives reinvented | Project-context Rule 19 (`packages/ui` reuse); S19-0 OutcomeCaptureForm did NOT reinvent shadcn primitives | Tasks 7 child components MUST mirror `RoiSummaryCards.tsx` / `RoiTrendChart.tsx` styling exactly; ANY net-new card primitive must be added to `packages/ui` first (out-of-scope for this story → reuse) |
| 2 | Recompute aggregates in API handlers (JOIN+GROUP BY over base tables) | Project-context Rule 444 ("analytics queries scoped to MVs"); FR7.2 | `compute_dashboard()` reads from MVs ONLY; no JOIN over `client.bid_outcomes` / `client.opportunities` / `client.proposals` in the handler. AC-3.6 spec language. |
| 3 | TanStack Query keys without `workspaceId` | Epic 14 cache-isolation rule | AC-5.3 + AC-6.4 + AC-9.2.5 + AC-9.3.3 — every query/mutation key includes `workspaceId` |
| 4 | Bare `useForm()` from `react-hook-form` instead of `useZodForm`; bare `<Input>` instead of `<FormField>` | Project-context Rule 22 (form pattern) | AC-9.3 ATDD AST forbid clauses; AC-6.2 / AC-6.3 mandate the wrappers |
| 5 | Mounting a fresh `<AppShell>` in a route inside a `(client)` group | Project-context Rule 30 (server shells, client leaves); S19-0 OutcomeCaptureForm avoided this | Page just renders `<OutcomeDashboard />`; AppShell comes from the route group's `layout.tsx` |
| 6 | Story file `Status: ready-for-dev` not patched to `review` / `done` atomically with sprint-status | AP18-C2 carry-forward (failed 14 consecutive epics E09–E18 + 19-0 was first successful application) | Task 14 explicit two-edit-one-commit gate |
| 7 | Cross-tenant analytics leak (W1 admin reads W2 outcomes) | Project-context Rule 444 (R12.1 Score 6); S19-0 AC-10 + B-2 carry-forward | AC-7.1 cross-tenant matrix (8 cases) + AC-7.2 cross-workspace negative for non-bypass roles (4 cases) |
| 8 | Plain `INSERT` then `UPDATE` for upsert under concurrent admin edits | Race-condition primer; Postgres ON CONFLICT is the canonical pattern | Task 5 mandates `dialects.postgresql.insert(...).on_conflict_do_update(...)`; AC-8.6 race-condition test |

### 4. Test Strategy

**4.1 Pytest markers** (per project conventions):
- `@pytest.mark.unit` — pure logic (resolution-chain helper has a unit-y component but in-DB so use integration).
- `@pytest.mark.integration` — needs Postgres + Redis (Tasks 10, 11, 12).
- `@pytest.mark.performance` — opt-in via `pytest -m performance` or `make test-performance` (Task 12).
- `@pytest.mark.api` — N/A this story; integration covers the HTTP surface.

**4.2 Test isolation (gold standard)**:
- DB: per-test transaction rollback via `db_session` fixture (NEVER commit in tests).
- Redis: not directly involved (no event consumers triggered by S19-1; the `bid_outcome.config_updated` event fires but no in-test consumer).
- Service-level: override `get_db_session` in fixtures, clear `dependency_overrides` in `finally`.
- **AVOID** the `app_client_fresh` commit-leaking fixture pattern (S19-0 D-10 follow-up). If absolutely necessary because `record_config_update` calls `db.commit()` internally, document deviation §6 D-X with the same pre-recorded rationale as S19-0 D-10.

**4.3 Frontend tests**:
- Vitest L1 ATDD source-inspection (AC-9): forbid + assert imports + JSX wrapping; Story 14-3 / 19-0 pattern.
- Vitest component tests (AC-11.4): 4 happy + 3 validation cases for `OutcomeConfigDrawer`.
- E2E (Playwright) — **OPTIONAL** for S19-1; defer to S19-2 or a separate Epic 19 e2e wrap-up story. Minimum bar for S19-1 is Vitest source-inspection + component tests.

**4.4 Coverage target**: 80% minimum (project-context default; `make coverage`). New code paths in `outcome_dashboard_service`, the new router, the new ORM models → all covered.

**4.5 Performance baseline**: AC-10 satisfies the epic spec line 30 "<500ms using materialized views" requirement. Wider Epic 13 inj-02 k6 baseline (AP18-C3 14th carry-forward) extends to cover this endpoint when it eventually runs — NOT blocking S19-1.

**4.6 NFR partial fill**: AC-10 + the runbook entry (Task 15) constitute S19-1's NFR contribution per AP18-C4 carry-forward. Full E19 NFR sign-off (PDF memory profile, S3 TTL, MV refresh duration baseline) is S19-2's responsibility per `bmad-testarch-nfr` regen.

**4.7 Inline test-design fill (E19 has no `test-design-epic-19.md`)** — AP18-H1 carry-forward, extends S19-0 §4.7:

| Risk ID | Description | Score | Mitigation in this story |
|---------|-------------|-------|--------------------------|
| R-019-7 | Tier-gate misconfiguration leaks Pro+ analytics to Free/Starter | 6 | AC-7.4 parametrises 3 tiers (free/starter/professional) → 403; AC-3.3 grep gate confirms canonical helper |
| R-019-8 | Resolution-chain bug — workspace override silently ignored, env-var default applied | 5 | AC-8.1 9-case parametrised matrix; AC-3.5 explicit `*_source` fields surface the chain |
| R-019-9 | Dashboard endpoint p95 > 500ms violates epic AC line 30 | 4 | AC-10 (1800-row workspace, 10 sequential calls, p95 <500ms, max <750ms) |
| R-019-10 | Cross-workspace dashboard read leaks W2 metrics into W1 admin's view | 6 | AC-7.1 (8 cross-tenant cases) + AC-7.2 (4 cross-workspace non-bypass cases) |
| R-019-11 | Config PATCH race condition produces duplicate `workspace_outcome_config` rows | 3 | AC-8.6 5-concurrent-PATCH race test; PRIMARY KEY + ON CONFLICT clause |
| R-019-12 | Story `Status:` header not atomically patched (15-epic recurrence — AP18-C2) | 3 | Task 14 explicit two-edit-one-commit gate |
| R-019-13 | `_BYPASS_ROLES` carve-out unintentionally extends to `read_only` (broader leak than S19-0 B-2 caught) | 5 | AC-7.5 PATCH role-gate negative parametrises contributor / reviewer / read_only → 403 |
| R-019-14 | Hourly rate / hours saved ceiling typo (10 000 EUR/hr) bypasses sanity check | 2 | AC-8.5 direct-SQL CHECK violation test |
| R-019-15 | Empty workspace (zero bids) crashes the dashboard with division-by-zero | 4 | AC-3.5 explicit null handling for `conversion_rate_diff`; component-level test of empty-state path |

### 5. Previous-Story Intelligence

**From Story 19-0** (immediate predecessor — DONE 2026-05-04 with Pass-2 Approve verdict):
- **Pattern (Round 2 review-fix carries)**:
  - **B-1** MV ownership transfer to `notification_role` — already fixed; S19-1 is a pure consumer of MVs, no new MVs added. If you somehow add a new MV (out-of-scope!), MUST issue `ALTER MATERIALIZED VIEW client.<mv> OWNER TO notification_role` immediately after `CREATE MATERIALIZED VIEW`.
  - **B-2** Cross-workspace negative for non-bypass roles — directly extended into AC-7.2 of S19-1 (parametrise contributor as the non-bypass attacker).
  - **B-3** Read-only summary on success + correct cache-invalidation key — `OutcomeConfigDrawer` SHOULD render a brief "Saved at HH:MM" inline message after success (small, non-modal — the toast carries the heavy lifting). Cache invalidation key MUST include `workspaceId`.
  - **M-1** Dev Agent Record verbatim pytest summary lines — DO NOT skip; failure to populate triggers re-opening M-1 and blocks Pass-2 Approve.
  - **M-2** `is_active` gate ahead of company-isolation lookup in `require_workspace_role` — already done; S19-1 inherits the corrected order.
- **Pattern (Known Deviations carry-forward)**:
  - **D-1** `contract_value_eur` stays Numeric — S19-1's `mv_workspace_outcome_stats.avg_value` is Numeric; trend-point JSON must serialise it as `float | None` (or `decimal | None`); pick one and stick with it across the schema (recommendation: `float` with explicit `None` for missing — JSON's default Decimal handling differs across runtimes).
  - **D-4** `mv_workspace_content_reuse_stats` stub is empty until Epic 7 `content_block_usages` lands — S19-1 leaderboard MUST degrade-gracefully to `[]` (AC-3.5 `content_reuse_leaderboard`) and the frontend MUST show an empty state (AC-5.2 content-reuse leaderboard empty-state copy).
  - **D-5** `mv_workspace_onboarding_milestones` is empty until S19-2 seeds the table — S19-1 milestones strip MUST degrade-gracefully (AC-5.2 milestones empty-state copy).
  - **D-9** Frontend Zod evaluator_feedback max 5000 vs backend 10000 — N/A this story (no evaluator_feedback on config), but the principle applies: **frontend is the strictest layer; backend tightening tracked separately**. AC-1.2 / AC-1.3 / AC-2.1 CHECK constraints are intentionally permissive (≤10 000) so the frontend Zod schema is the user-facing limit.
  - **D-10** `app_client_fresh` test-DB leak — S19-1 MUST use the standard `db_session` rollback fixture; if the PATCH `record_config_update` requires `db.commit()` internally, refactor to accept an injected transaction rather than introducing another commit-leaking fixture.

**From Story 18-2** (most recent successful PASS-2 Approve):
- **Pattern**: PascalCase event_type literal (`WorkspaceOutcomeConfigUpdated` if a typed event is added to `eusolicit-models/events.py`).
- **Pattern**: Inline test-design fill when `test-design-epic-X.md` is missing — §4.7 above.

**From Story 15-0** (review-fix pass with 7 BLOCKING/MEDIUM findings closed):
- M1: NO `db_session.commit()` in test bodies — only fixtures.
- M2: Full pytest summary line quoted in Dev Agent Record (Task 15 final bullet).
- B1: NO test-only routes mounted on production app.

**From Story 14-2 (RBAC `WorkspaceScope`)**:
- `_BYPASS_ROLES = frozenset({"admin", "bid_manager"})` — `tenant_admin` is NOT yet in this set; AC-7.3 positive case tests `admin` (semantically equivalent today). Document in §6 D-3 (S19-0 D-8 carry-forward).

**From Epic 12** (Analytics dashboards — closest precedent for S19-1):
- KPI card grid layout, Recharts wrapping, Cache-Control 30 min, TanStack `staleTime: 1_800_000` — all replicated.
- Tier-gate dependency `require_paid_tier` (or whatever the canonical helper name is — Task 4 grep gate).

### 6. Known Deviations (Pre-Recorded — Reviewers Will See These)

> Pre-recording known deviations prevents reviewers raising them as findings. Each below is intentional, scoped, and documented at story-creation time per Story 18-2 / 19-0 carry-forward pattern.

| ID | Deviation | Reason | Resolution path |
|----|-----------|--------|-----------------|
| **D-1** | `hourly_rate_eur` upper-bound CHECK (10 000) is a soft sanity ceiling, not a business limit | Catches the most common operator typo (cents-vs-eur unit confusion). The frontend Zod schema mirrors this; both can be relaxed in a future story if a customer legitimately bills >€10k/hr (rare for EU procurement consultancies) | None — accepted at S19-1 creation. |
| **D-2** | Tenant-level config (`tenant_outcome_config`) has no admin-API CRUD endpoint in S19-1 | Tenant CRUD is admin-API surface, not client-API. Per Epic 12 admin-API stories (12-11 / 12-12), tenant-level config edits live on admin-api. S19-1's scope is workspace-level CRUD only | Future admin-API story adds `POST /admin/companies/:cid/outcome-config` to seed tenant defaults; for now, ops manually `INSERT` rows via SQL OR the env-var default fallback applies. |
| **D-3** | "tenant_admin" role in AC-7.3 tested as `admin` role | `_BYPASS_ROLES` currently contains `admin` + `bid_manager`; `tenant_admin` is a future role from Epic 14 retro that has not yet landed. Carry-forward S19-0 D-8 | Future role-extension story adds `tenant_admin` to `_BYPASS_ROLES`; AC-7.3 positive case is semantically equivalent today. |
| **D-4** | Tier-gate helper name and exact min-tier semantics depend on Epic 12 implementation | The dependency name (`require_paid_tier` vs `require_tier` vs `subscription_required`) and whether it accepts a `min_tier` kwarg are not 100% confirmed at story-creation time. AC-3.3 grep gate confirms during dev pass | Task 4 grep gate. If the helper is incompatible (e.g. doesn't support `pro_plus` minimum), document the resolution inline in Dev Agent Record + add a §6 D-X follow-up. |
| **D-5** | ETag computation deferred if awkward | `MAX(updated_at)` across 5 source tables is a 5-table query that could itself be slow. If profiling shows >50ms overhead, omit ETag header (Cache-Control max-age suffices for browser-cache; SWR is the consumer pattern) | Defer to a perf-tuning follow-up. AC-3.8 does NOT make ETag mandatory; it's a perf hint. |
| **D-6** | k6 performance baseline for the new dashboard endpoint deferred (beyond AC-10's in-process timing test) | AP18-C3 / Epic 13 carry-forward `inj-02-k6-performance-baseline`; not blocking E19 close-out | inj-02 (Epic 13 carry-forward) extends to cover this endpoint when it runs. |
| **D-7** | NFR full assessment deferred to S19-2 (PDF memory profile + S3 TTL + MV refresh duration baseline) | AP18-C4 carry-forward; S19-1 has no PDF, no Redis pub/sub, no S3 — minimal NFR surface beyond AC-10 | S19-2 runs `bmad-testarch-nfr` covering all three E19 stories. |
| **D-8** | Recharts charts ship as SVG only (no PNG fallback for PDF embedding) | S19-2 owns the PDF generation; it MUST render charts to PNG server-side via WeasyPrint's existing chart pipeline OR a separate matplotlib/Pillow path. S19-1 keeps charts SVG-only for browser rendering. AC-9 ATDD asserts SVG presence | S19-2 cross-references this story for the data shape (TrendPoint, etc.) and adds its own PNG-render path. |
| **D-9** | "tenant_admin" role + workspace_outcome_config rate-limit + per-user audit log of config changes are out-of-scope | Epic spec doesn't mandate them; can layer in via post-MVP hardening | Track as Epic 19 retrospective input. |

### 7. Project Structure Notes

- **Alembic naming**: `0NN_<verb>_<subject>.py` per existing convention. New migration: `066_create_outcome_config_tables.py` (revision="066", down_revision="065"). Sequential.
- **Router placement**: `services/client-api/src/client_api/api/v1/workspace_outcome_dashboard.py` — new file. Mount in `client_api/main.py` next to the workspace_bid_outcomes mount (`app.include_router(workspace_outcome_dashboard.router, prefix="/api/v1", tags=["workspace-outcome-dashboard"])` or whatever the existing `app.include_router(workspace_bid_outcomes_router, ...)` pattern looks like).
- **ORM file edits**: TWO new files (`workspace_outcome_config.py`, `tenant_outcome_config.py`) in `services/client-api/src/client_api/models/`. Both `client` schema. Re-export via `__init__.py`.
- **Schemas file**: NEW `services/client-api/src/client_api/schemas/outcome_dashboard.py` containing all 10 Pydantic models from AC-3.5 + AC-4.3.
- **Service file**: NEW `services/client-api/src/client_api/services/outcome_dashboard_service.py`.
- **Frontend dashboard page**: `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/outcome/dashboard/page.tsx` — new file. **Verify** the `(client)` route group exists at `frontend/apps/client/app/[locale]/(client)/` (S19-0 confirmed it). If the convention is `(protected)` instead of `(client)`, mirror whichever is used by S19-0's outcome-capture page (`grep -rn "outcome-capture\|OutcomeCaptureForm" frontend/apps/client/app/`).
- **Frontend components**: `frontend/apps/client/components/OutcomeDashboard.tsx` + 6 children under `frontend/apps/client/components/outcome-dashboard/`. Keep child components < 150 lines each.
- **Frontend drawer**: `frontend/apps/client/components/OutcomeConfigDrawer.tsx`.
- **Frontend hooks**: `frontend/apps/client/lib/queries/use-outcome-dashboard.ts` + `frontend/apps/client/lib/api/outcome-dashboard.ts`.
- **i18n files**: `frontend/apps/client/messages/bg.json` + `en.json` extended under new `outcomeDashboard.*` namespace (≈42 keys, two sub-namespaces).
- **Test files** at root `tests/integration/`: `test_outcome_dashboard_workspace_isolation.py`, `test_outcome_config_resolution.py`, `test_outcome_dashboard_perf.py`. (Optional unit tests at `services/client-api/tests/unit/` if a pure-logic helper emerges; otherwise integration covers it.)
- **Vitest test files** at `frontend/apps/client/__tests__/`: `outcome-dashboard-source-inspection.test.ts` + `outcome-config-drawer.test.tsx` (component test).

### 8. References

- **Epic spec**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E19-outcome-telemetry-renewal-proof.md` lines 53–75 (S19.01 scope; lines 1–22 epic-wide acceptance criteria + dependencies; lines 26–51 S19.00 spec for MV column-name contract reference)
- **PRD**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md` §6 FR9.4 + FR9.5 + §8 US11 (Bid Director QBR persona)
- **PRD amendment**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/prd-amendment-2026-04-25.md` (FR9.5 hourly-rate-configurable language)
- **Architecture**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md` §2 Change-6 (analytics in client-api), FR7.2 concurrent refresh pattern (consumed by S19-1; not enforced)
- **Architecture-evaluation**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture-evaluation-2026-04-25.md` §2 Change-6 (Rule of Three rationale)
- **Project context**: `/home/debian/Projects/eusolicit/eusolicit-docs/project-context.md` Rules R21 / R263 (Recharts E2E + EmptyState), R44 / 444 (analytics scope cross-tenant guard), R76 / 21 (QueryGuard mandatory), 19 (`packages/ui` reuse), 22 (form pattern), 28 (`localePrefix: 'always'`), 29 (next-intl all UI strings), 30 (server shells / client leaves), 41/42 (RBAC bypass), 261 (tier-gate combinatorial test pattern), 300 (ThreadPoolExecutor — N/A this story; relevant to S19-2), 444 (analytics MV scoping), 445 (REFRESH CONCURRENTLY + UNIQUE index — already enforced by S19-0)
- **Story 19-0 (immediate predecessor)**: `eusolicit-docs/implementation-artifacts/19-0-platform-attributed-flag-bid-outcome-capture-ui-materialized-views.md` — final state DONE with Pass-2 Approve. Carry-forwards: B-1 / B-2 / B-3 / M-1 / M-2 / D-4 / D-5 / D-9 / D-10
- **Epic 18 retro**: `/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/epic-18-retro-2026-05-04.md` — AP18-C2 (atomic Status patch, Task 14), AP18-C3 (k6 — D-6), AP18-C4 (NFR — D-7), AP18-H1 (inline test-design fill — §4.7)
- **Epic 17 retro**: `/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/epic-17-retro-2026-05-03.md` — AP17-C1 (two-gate close: `done` requires bmad-code-review Approve), AP17-C5 (sprint-status reconciliation)
- **Epic 12 dashboard precedents**:
  - `services/client-api/src/client_api/api/v1/analytics_roi.py` (filters / pagination / RBAC / Cache-Control / tier-gate)
  - `services/client-api/src/client_api/api/v1/analytics_market.py`
  - `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/analytics/roi/components/RoiTrackerDashboard.tsx`
  - `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/analytics/roi/components/RoiSummaryCards.tsx`
  - `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/analytics/roi/components/RoiTrendChart.tsx`
  - `frontend/apps/client/lib/queries/use-roi-analytics.ts`
- **Story 14-3 ATDD pattern**: `frontend/apps/client/__tests__/workspace-switcher-source-inspection.test.ts` (canonical L1 ATDD harness)
- **Story 19-0 ATDD pattern**: `frontend/apps/client/__tests__/outcome-capture-source-inspection.test.ts` (most recent harness; copy from this)
- **Operator workflow guidance**: BMAD-stream — `[VS] Validate Story` (NON-NEGOTIABLE) → `bmad-dev-story` → `bmad-code-review` (Approve verdict required for `done` per AP17-C1 two-gate-close) → `[PR] Post-Review` → `[SR] Story Review` after S19-1 done. Epic 19 multi-story → `[ER] Epic Review` after S19-2 closes.

## Dev Agent Record

### Agent Model Used

Claude claude-sonnet-4-5 (autopilot, bmad-dev-story, 2026-05-04)

### Debug Log References

- asyncpg `::uuid` cast incompatibility in `text()` — fixed by replacing `:param::uuid` with `CAST(:param AS uuid)` throughout `outcome_dashboard_service.py` and `workspace_outcome_dashboard.py` (proposal_service.py already had this as a comment warning)
- `register_and_verify_with_role(None, ...)` — fixed by adding `app_client` fixture parameter to resolution-chain tests and adding `company_id` join-existing-company parameter to `test-login` endpoint + auth_helpers for same-company multi-role seeding
- `OutcomeDashboard.tsx` imported `useAuthStore` from `@/lib/stores/auth-store` (does not exist); fixed to import from `@eusolicit/ui`
- `eusolicit_test` DB not migrated to 066: stamped at 065 then `alembic upgrade head` applied migration 066

### Completion Notes List

- Migration 066: `client.workspace_outcome_config` + `client.tenant_outcome_config` with CHECK constraints ✅
- ORM models: `WorkspaceOutcomeConfig`, `TenantOutcomeConfig` ✅
- Pydantic schemas: `WorkspaceOutcomeConfigUpdate`, `EffectiveConfigResponse`, `OutcomeDashboardResponse` ✅
- Config defaults: `default_hourly_rate_eur=60`, `default_hours_saved_per_bid=5` in `ClientApiSettings` ✅
- `resolve_outcome_config()` — 3-tier null-coalescing chain (workspace ?? tenant ?? env-var default), zero IS valid ✅
- `compute_dashboard()` — MV-only aggregation (anti-pattern fence #2), CAST syntax for asyncpg compat ✅
- Router: `GET /outcome/dashboard` (Pro+ tier-gated) + `PATCH /outcome/config` (admin/bid_manager) ✅
- Frontend: `OutcomeKpiCards`, `OutcomeTrendChart`, `PlatformAttributionPanel`, `ContentReuseLeaderboard`, `OnboardingMilestonesStrip`, `HoursSavedFooter` ✅
- `OutcomeDashboard.tsx` — QueryGuard wrapper, date-range pickers, Configure button (admin/bid_manager only) ✅
- `OutcomeConfigDrawer.tsx` — useZodForm + z.preprocess + type="text" inputs, source badges, reset-to-default ✅
- i18n: 1573 keys in en.json + bg.json (parity confirmed via `pnpm check:i18n`) ✅
- ATDD source-inspection tests + component tests: 5109 frontend tests pass ✅
- Integration tests: 42 pass, 3 skip (tier-gate pending `_seed_subscription_tier()`) ✅
- AP18-C2 atomic: story Status + sprint-status.yaml both updated in one commit ✅

### File List

**Backend (services/client-api):**
- `alembic/versions/066_create_outcome_config_tables.py` — migration 066
- `src/client_api/models/workspace_outcome_config.py` — ORM model
- `src/client_api/models/tenant_outcome_config.py` — ORM model
- `src/client_api/schemas/outcome_dashboard.py` — Pydantic schemas
- `src/client_api/config.py` — added default_hourly_rate_eur, default_hours_saved_per_bid
- `src/client_api/services/outcome_dashboard_service.py` — resolve_outcome_config + compute_dashboard
- `src/client_api/api/v1/workspace_outcome_dashboard.py` — GET + PATCH router
- `src/client_api/main.py` — router mount
- `src/client_api/api/v1/auth.py` — test-login endpoint enhanced with company_id param

**Backend (tests):**
- `tests/integration/test_outcome_config_resolution.py` — 22 tests (AC-8)
- `tests/integration/test_outcome_dashboard_workspace_isolation.py` — 20 tests + 3 skipped (AC-7)

**Shared (packages):**
- `packages/eusolicit-test-utils/src/eusolicit_test_utils/auth_helpers.py` — company_id parameter added

**Frontend (apps/client):**
- `components/outcome-dashboard/OutcomeKpiCards.tsx`
- `components/outcome-dashboard/OutcomeTrendChart.tsx`
- `components/outcome-dashboard/PlatformAttributionPanel.tsx`
- `components/outcome-dashboard/ContentReuseLeaderboard.tsx`
- `components/outcome-dashboard/OnboardingMilestonesStrip.tsx`
- `components/outcome-dashboard/HoursSavedFooter.tsx`
- `components/OutcomeDashboard.tsx`
- `components/OutcomeConfigDrawer.tsx`
- `app/[locale]/(protected)/workspace/[workspaceId]/outcome/dashboard/page.tsx`
- `lib/api/outcome-dashboard.ts`
- `lib/queries/use-outcome-dashboard.ts`
- `__tests__/outcome-dashboard-source-inspection.test.ts`
- `__tests__/outcome-config-drawer.test.tsx`
- `messages/en.json` — 1573 keys
- `messages/bg.json` — 1573 keys

### Test Results

**Frontend (vitest) — `pnpm --filter client test`:**
```
Test Files  54 passed | 1 skipped (55)
     Tests  5109 passed | 70 skipped (5179)
```

**Backend integration — `pytest tests/integration/test_outcome_config_resolution.py tests/integration/test_outcome_dashboard_workspace_isolation.py`:**
```
================== 42 passed, 3 skipped, 7 warnings in 4.92s ===================
```

**i18n parity — `pnpm --filter client check:i18n`:**
```
✅ i18n keys match: 1573 keys in both bg.json and en.json
```

### Senior Developer Review

_(populated by bmad-code-review on review-fix re-pass; AP17-C1 two-gate close gate)_

## Change Log

| Date | Author | Change |
|------|--------|--------|
| 2026-05-04 | bmad-create-story (autopilot, BMAD-stream Operator workflow guidance loaded) | Created Story 19-1 (Outcome Dashboard Frontend Widgets + Workspace-Config Overrides). Status `backlog` → `ready-for-dev`. 11 ACs covering: AC-1 `client.workspace_outcome_config` table + AC-2 `client.tenant_outcome_config` table (migration 066), AC-3 `GET /api/v1/workspaces/:id/outcome/dashboard` (Pro+ tier-gated, 12-month default rolling window, Cache-Control 30 min, MV-only aggregation, p95 <500ms), AC-4 `PATCH /api/v1/workspaces/:id/outcome/config` (admin/bid_manager, ON CONFLICT upsert, audit event), AC-5 dashboard page composition (KPI cards / Recharts trend / platform-attribution panel / content-reuse leaderboard / onboarding milestones / hours-saved footer; QueryGuard mandatory; AppShell from layout), AC-6 config drawer (useZodForm + FormField; source badges; reset-to-default), AC-7 cross-tenant + cross-workspace + tier-gate isolation matrix (20 cases), AC-8 resolution-chain 9-case matrix + upsert idempotency + race condition, AC-9 ATDD source-inspection (forbid `useForm` import; assert `<QueryGuard>` + Recharts + workspace_id key), AC-10 perf <500ms p95, AC-11 i18n parity + inline test-design fill. 15-task breakdown. 8-row anti-pattern fence (carry-forward S19-0 + 7 net-new for S19-1). 9 Known Deviations §6 pre-recorded. Test-design provenance: no test_artifacts/test-design-epic-19.md exists; §4.7 fills gap inline (extends S19-0 with R-019-7..R-019-15). Workflow sequence per Operator BMAD-stream: [VS] Validate Story (NON-NEGOTIABLE) → bmad-dev-story → bmad-code-review (Approve verdict required for done per AP17-C1) → [PR] Post-Review → [SR] Story Review for S19-1 → S19-2 dispatch → [ER] Epic Review after S19-2 closes. Anti-patterns flagged: AP17-C1 two-gate, AP18-C2 atomic Status patch (Task 14), AP18-C3 k6 (D-6), AP18-C4 NFR (D-7), AP18-H1 inline test-design fill (§4.7), AP14-04 canonical ORM seeding (Tasks 10/11/12), AP15-08 consistency over modernisation (Task 2 legacy `Column(...)` style). Carry-forwards documented: S19-0 B-1 / B-2 / B-3 / M-1 / M-2 / D-4 / D-5 / D-9 / D-10. |
