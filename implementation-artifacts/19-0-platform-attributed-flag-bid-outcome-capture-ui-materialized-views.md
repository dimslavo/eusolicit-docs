# Story 19.0: Platform-Attributed Flag + Bid-Outcome Capture UI + Materialized Views

Status: review

<!-- Validation note: Run [VS] Validate Story (bmad-validate-story) before bmad-dev-story per Operator BMAD-stream Operator workflow guidance — non-negotiable. AP18-C2 carry-forward: orchestrator MUST patch `Status: done` in this file atomically with sprint-status transition (failed 13 consecutive epics E09–E18). -->

## Story

As a **Bid Manager (and downstream Bid Director consuming the Outcome Dashboard)**,
I want **to log the outcome of a submitted opportunity (won / lost / withdrawn) with evaluator score, contract value and effort hours, AND have the platform automatically tag opportunities sourced from the platform feed so they distinguish from externally-uploaded leads**,
so that **my workspace builds an accurate, materialized-view-backed historical record of platform-attributed win-rate, content-reuse and hours-saved metrics that the Outcome Dashboard (Story 19.1) and Monthly Outcome Brief PDF (Story 19.2) consume to prove ROI for renewal.**

## Epic Context

- **Epic**: E19 Outcome Telemetry & Renewal Proof (Sprint 17–18; 21 pts; Milestone: Renewal engine)
- **Story points**: 8 | **Type**: backend + frontend (alembic migration + FastAPI route extension + Postgres materialized views + Next.js form)
- **FRs covered**: FR9.1, FR9.2, FR9.3 (outcome capture, platform attribution, materialized-view-backed metrics) — partial coverage of FR9.4 / FR9.5 (PDF + dashboard widgets) deferred to S19.1 / S19.2
- **Position in epic chain**: Foundational story of Epic 19. Hard-blocks S19.1 (Outcome Dashboard reads `mv_workspace_outcome_stats`) and S19.2 (Monthly Outcome Brief PDF reads `mv_workspace_outcome_stats` + `mv_workspace_onboarding_milestones`). The MV column names + types defined here are the public contract S19.1/S19.2 depend on (per E18 retro lesson "interface contracts must be explicit").
- **Source**: Epic spec `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E19-outcome-telemetry-renewal-proof.md` lines 26–51 (S19.00). PRD v1.1 §6 FR9 + §8 US11. Architecture-evaluation §2 Change-6 (analytics stays in client-api per Rule of Three; do NOT split into a new service).

## Acceptance Criteria

> Source-of-truth: epic spec lines 26–51. AC numbers below cover every epic line item plus carry-forward hardening from Epic 14/15/16/17/18 retros (AP14-04 canonical ORM seeding; AP15-08 cross-tenant parametrisation; AP17-C1 two-gate close; AP18-C1 deferred WeasyPrint hardening NOT a blocker for S19.0 since this story has no PDF; AP18-C2 atomic Status patch; AP18-H1 inline test-design fill).

### AC-1 — `client.opportunities.platform_attributed` Column

**Given** the existing `client.opportunities` table (created by migration 056),
**When** alembic migration `063_add_platform_attributed_to_opportunities.py` runs,
**Then**:
1. Column `platform_attributed BOOLEAN NOT NULL DEFAULT FALSE` is added (server_default `false` so existing rows backfill to FALSE without a multi-step migration).
2. A non-unique index `ix_opportunities_workspace_platform_attributed` on `(workspace_id, platform_attributed)` is created — required for the `mv_workspace_outcome_stats` daily refresh predicate (`WHERE platform_attributed = true`).
3. Migration is reversible (`downgrade()` drops the index then the column).
4. The `Opportunity` ORM model at `services/client-api/src/client_api/models/opportunity.py` is updated to declare `platform_attributed: Mapped[bool] = mapped_column(sa.Boolean, nullable=False, server_default=sa.text("false"))`. `__table_args__` index list is extended to include the new index (do NOT add a second `Index(...)` call — keep all in `__table_args__` per the file's existing convention).
5. Anti-pattern guard: do NOT add `comment="..."` to ORM column unless the test asserts it round-trips through `alembic check` (project-context Rule R444 — migration spec contradiction). If you add a comment, add the alembic test that asserts schema parity.

### AC-2 — Auto-Set `platform_attributed=TRUE` on Pipeline-Sourced Ingest

**Given** the data-pipeline service ingests opportunities from KraftData and writes them into `pipeline.opportunities`,
**When** a workspace user "imports" or "tracks" a pipeline opportunity into their workspace (creating a corresponding `client.opportunities` row),
**Then**:
1. The ingest path that copies a `pipeline.opportunities` row into `client.opportunities` MUST set `platform_attributed = TRUE` on the new row.
2. The external-upload path (user uploads a tender PDF, manual create flow, CRM-sourced row from Story 17.1) MUST leave `platform_attributed = FALSE` (default).
3. Find the existing copy/ingest code path:
   - Search `services/client-api/src/client_api/services/` for the function that creates a `client.opportunities` row from a `pipeline.opportunities` row (likely named `track_opportunity`, `import_opportunity`, or `link_pipeline_opportunity`).
   - If no such service exists yet, the Story 12.2 / 12.3 flow may write directly to `client.opportunities`; in that case, set `platform_attributed=TRUE` at the write site identified by `crm_external_provider IS NULL AND <some pipeline-origin marker>`.
   - If neither exists, document the gap as Known Deviation §6 D-2 ("auto-set deferred to S19.1; default-FALSE preserves correctness, just under-counts platform-attributed bids until backfill"). DO NOT block this story on a missing ingest path.
4. Manual-upload / CRM-sourced rows: `Opportunity.crm_external_provider IS NOT NULL` → `platform_attributed = FALSE` is the invariant (CRM-sourced is by definition externally attributed, not platform-attributed).
5. Unit test: parametrised over `source ∈ {pipeline_feed, manual_upload, crm_hubspot, crm_pipedrive, crm_salesforce}` asserting the resulting `platform_attributed` value.

### AC-3 — `client.bid_outcomes` Schema Extension (`evaluator_score`, `effort_hours`)

**Given** the existing `client.bid_outcomes` table (Migration 011 base + Migration 045 runtime extensions; ORM at `services/client-api/src/client_api/models/bid_outcome.py`),
**When** alembic migration `064_extend_bid_outcomes_for_e19_capture.py` runs,
**Then**:
1. NEW column `evaluator_score INTEGER` (nullable) — single-number score (0–100 typical EU procurement scale). This is **distinct from the existing `evaluator_scores JSONB` column**, which holds per-criterion scoring (technical / financial / experience). Add a `CheckConstraint("evaluator_score IS NULL OR (evaluator_score >= 0 AND evaluator_score <= 100)", name="ck_bid_outcomes_evaluator_score_range")` — fails-closed on out-of-range writes.
2. NEW column `effort_hours INTEGER` (nullable) — operator-supplied effort estimate in hours, used by the `mv_workspace_outcome_stats.hours_saved_estimate` calculation in S19.1. Add a `CheckConstraint("effort_hours IS NULL OR effort_hours >= 0", name="ck_bid_outcomes_effort_hours_nonneg")`.
3. **Known Deviation D-1**: Epic spec line 31 says `contract_value_eur INTEGER`. The column **already exists as `Numeric` (nullable)** in Migration 011 + ORM `bid_outcome.py:45`. Story 19.0 does NOT migrate Numeric → Integer (would require historical data scan + risk truncation). Document in §6 D-1 that the column type stays `Numeric` and the API accepts `int | float` and casts on read for MV compatibility (`(contract_value_eur)::bigint AS contract_value_eur` in MV definitions).
4. ORM `BidOutcome` updated: add `evaluator_score: Mapped[int | None]` and `effort_hours: Mapped[int | None]` with appropriate `Mapped`/`mapped_column` syntax matching the rest of the file (note: file currently uses **legacy `Column(...)` syntax**, NOT modern `Mapped[...]` — match the existing style; do NOT mix syntaxes within the file. AP15-08 carry-forward: consistency over modernisation churn).
5. `BidOutcomeCreateRequest` schema at `services/client-api/src/client_api/schemas/bid_outcomes.py` extended with `evaluator_score: int | None = Field(None, ge=0, le=100)` and `effort_hours: int | None = Field(None, ge=0)`.
6. `BidOutcomeResponse` schema extended with the two new fields (read-back in API response).

### AC-4 — Workspace-Scoped Outcome POST Endpoint

**Given** the existing endpoint `POST /api/v1/opportunities/{opportunity_id}/outcome` at `services/client-api/src/client_api/api/v1/bid_outcomes.py` (Story 10.11; company-scoped, NOT workspace-scoped),
**When** Story 19.0 lands,
**Then**:
1. A NEW workspace-scoped path is added: `POST /api/v1/workspaces/{workspace_id}/opportunities/{opportunity_id}/outcome`.
   - Mounted on the existing `bid_outcomes` router OR a new router at `services/client-api/src/client_api/api/v1/workspace_bid_outcomes.py` — pick whichever keeps router prefix consistency (recommendation: NEW router with `prefix="/workspaces/{workspace_id}/opportunities/{opportunity_id}/outcome"`, tags `["workspace-bid-outcomes"]`).
2. The new endpoint validates that `client.opportunities.id == opportunity_id AND client.opportunities.workspace_id == workspace_id` — returns **404** (NOT 403, to avoid information leak per AP14-04 cross-tenant enumeration anti-pattern) when the opportunity is not in the path's workspace.
3. RBAC: `Depends(require_role("bid_manager"))` (matches existing endpoint). Plus: workspace-membership check — current user MUST belong to a `client_workspaces` row whose `id == workspace_id` (see Epic 14 RBAC `WorkspaceScope` dependency). Cross-workspace bypass for `tenant_admin` role per Epic 14 carry-forward (`_BYPASS_ROLES` in `services/client-api/src/client_api/core/rbac.py:102–114`).
4. The existing route `POST /api/v1/opportunities/{opportunity_id}/outcome` is **retained** for backward compatibility with Story 10.11 callers (frontend Outcome capture form will use the NEW workspace-scoped path; legacy clients keep working). Add a `@deprecated` docstring + log-warning `bid_outcome.deprecated_path_used` on the legacy route. Do NOT delete the route in this story (AP15-08 carry-forward: deletion is its own story, never a side-effect of feature work).
5. Service layer: extend `bid_outcome_service.record_outcome(...)` with optional `workspace_id: UUID | None = None` kwarg. When provided, the service additionally validates `client.opportunities.workspace_id == workspace_id` BEFORE creating the outcome row (raise `HTTPException(404, "Opportunity not found in this workspace")` on mismatch).
6. The existing service path that validates against `pipeline.opportunities` (`services/client-api/src/client_api/services/bid_outcome_service.py:46`) — leave unchanged for backward-compat. The NEW workspace-scoped service path validates against `client.opportunities` (workspace-scoped table). Document in §6 D-3 that this introduces a temporary dual lookup; convergence on `client.opportunities` is deferred to Story 19.1 cleanup.

### AC-5 — Outcome Capture Frontend Form

**Given** a Bid Manager viewing an opportunity in workspace W,
**When** they navigate to the opportunity detail page after marking a proposal as "submitted",
**Then**:
1. An "Outcome Capture" panel renders inline on the opportunity detail page (NOT a modal — modal flows hide the form from screenshots / e2e testing per project-context Rule R263).
2. The form uses `useZodForm(schema)` from `@eusolicit/ui` (frontend/packages/ui/src/lib/hooks/useZodForm.ts) — **NOT** direct `useForm()` from `react-hook-form`. **ATDD source-inspection** asserts the import line. Hard-fail anti-pattern: a bare `import { useForm } from "react-hook-form"` in this component is a CI failure.
3. Form fields:
   - `outcome` — RadioGroup of `won` | `lost` | `withdrawn` (3 options; required).
   - `proposal_id` — hidden field, prefilled from the opportunity's submitted proposal (bid_manager picks the proposal from a select if multiple submitted).
   - `evaluator_score` — Number input, optional, range 0–100.
   - `contract_value_eur` — Number input (currency-formatted on blur), optional, ≥ 0.
   - `effort_hours` — Number input, optional, ≥ 0.
   - `evaluator_feedback` — Textarea, optional, max 5000 chars (existing pattern; matches `evaluator_feedback Text` column).
4. Each field is wrapped in `<FormField name="..." label="..." />` from `@eusolicit/ui` (frontend/packages/ui/src/components/forms/FormField.tsx) — **NOT** a direct `<Input />` + `<Label />` pair. ATDD source-inspection asserts every input is inside a `<FormField>`.
5. Submit handler calls the new workspace-scoped path `POST /api/v1/workspaces/{workspace_id}/opportunities/{opportunity_id}/outcome` via the shared API client `frontend/apps/client/lib/api/bid-outcomes.ts` (extend existing `recordBidOutcome()` with a `workspaceId` parameter; preserve backward-compat by making it optional and routing to the legacy path when omitted).
6. Success toast "Outcome recorded" + form replaces with read-only summary (the recorded outcome). Failure toast surfaces the API error message (NOT a generic "Something went wrong" — project-context Rule R76 / QueryGuard).
7. i18n: All labels, errors, and toasts have BG + EN translations. `pnpm check:i18n` passes (1502 keys parity baseline from Story 18-0; Story 19.0 adds ~12 new keys per locale).
8. The form is wrapped in `<QueryGuard>` for the proposal-list fetch (existing pattern; project-context Rule R76 — no scattered `if (isLoading) return <Spinner />`).
9. Cache invalidation: on success, `queryClient.invalidateQueries({ queryKey: ["opportunity", workspaceId, opportunityId] })` AND `["bid-outcome", workspaceId, opportunityId]`. TanStack Query keys MUST include `workspace_id` per Epic 14 cache-isolation rule.

### AC-6 — `mv_workspace_outcome_stats` Materialized View (Daily Refresh)

**Given** the new MVs in `client` schema,
**When** alembic migration `065_create_workspace_outcome_materialized_views.py` runs,
**Then**:
1. View definition:
   ```sql
   CREATE MATERIALIZED VIEW client.mv_workspace_outcome_stats AS
   SELECT
       o.workspace_id                                                   AS workspace_id,
       date_trunc('month', bo.created_at)::date                          AS month,
       COUNT(DISTINCT bo.opportunity_id)                                  AS bids_tracked,
       COUNT(DISTINCT CASE WHEN p.status = 'submitted' THEN bo.opportunity_id END) AS bids_submitted,
       COUNT(DISTINCT CASE WHEN bo.status = 'won' THEN bo.opportunity_id END)      AS bids_won,
       CASE
         WHEN COUNT(DISTINCT CASE WHEN p.status = 'submitted' THEN bo.opportunity_id END) = 0 THEN 0.0
         ELSE COUNT(DISTINCT CASE WHEN bo.status = 'won' THEN bo.opportunity_id END)::numeric
              / COUNT(DISTINCT CASE WHEN p.status = 'submitted' THEN bo.opportunity_id END)::numeric
       END                                                              AS win_rate,
       COUNT(DISTINCT CASE WHEN o.platform_attributed = TRUE THEN bo.opportunity_id END) AS platform_attributed_count,
       AVG(bo.contract_value_eur)::numeric                              AS avg_value
   FROM client.bid_outcomes bo
   JOIN client.opportunities o     ON o.id = bo.opportunity_id
   JOIN client.proposals p         ON p.id = bo.proposal_id
   GROUP BY o.workspace_id, date_trunc('month', bo.created_at)::date;
   ```
2. **UNIQUE index** on `(workspace_id, month)` — **REQUIRED** for `REFRESH MATERIALIZED VIEW CONCURRENTLY` (Postgres limitation; project-context Rule R21). Without this index, the daily refresh will fail. Index name: `ux_mv_workspace_outcome_stats_workspace_month`.
3. Ownership transferred to `notification_role` (per migration 011 lines 306–313 pattern; mandatory for `notification_role` to call `REFRESH MATERIALIZED VIEW CONCURRENTLY`):
   ```sql
   ALTER MATERIALIZED VIEW client.mv_workspace_outcome_stats OWNER TO notification_role;
   GRANT SELECT ON client.mv_workspace_outcome_stats TO client_role;
   ```
4. Read grant for `client_role` (S19.1 dashboard query consumer) — explicit `GRANT SELECT` after ownership transfer.
5. Daily refresh job: ADD a Celery Beat task in `services/notification/src/notification/celery_app.py` (or wherever migration 011's existing refresh tasks live — search for `REFRESH MATERIALIZED VIEW` in `services/notification/`) that runs `REFRESH MATERIALIZED VIEW CONCURRENTLY client.mv_workspace_outcome_stats` daily at 03:00 UTC. If the existing migration-011 refresh job already iterates over a list of MVs, just append the three new MV names to that list; do NOT add a new task.

### AC-7 — `mv_workspace_content_reuse_stats` Materialized View (Daily Refresh)

**Given** content-block usage tracking exists from Epic 7 (search for `content_blocks` and `content_block_usages` tables; ORM `services/client-api/src/client_api/models/content_block*.py`),
**When** the same alembic migration runs,
**Then**:
1. View definition:
   ```sql
   CREATE MATERIALIZED VIEW client.mv_workspace_content_reuse_stats AS
   SELECT
       cb.workspace_id          AS workspace_id,
       cb.id                    AS content_block_id,
       COUNT(cbu.id)            AS usage_count,
       CASE
         WHEN COUNT(bo_won.id) + COUNT(bo_lost.id) = 0 THEN 0.0
         ELSE COUNT(bo_won.id)::numeric / NULLIF((COUNT(bo_won.id) + COUNT(bo_lost.id)), 0)::numeric
       END                       AS win_rate_when_used
   FROM client.content_blocks cb
   LEFT JOIN client.content_block_usages cbu ON cbu.content_block_id = cb.id
   LEFT JOIN client.proposals p              ON p.id = cbu.proposal_id
   LEFT JOIN client.bid_outcomes bo_won      ON bo_won.proposal_id = p.id AND bo_won.status = 'won'
   LEFT JOIN client.bid_outcomes bo_lost     ON bo_lost.proposal_id = p.id AND bo_lost.status = 'lost'
   GROUP BY cb.workspace_id, cb.id;
   ```
2. UNIQUE index on `(workspace_id, content_block_id)` named `ux_mv_workspace_content_reuse_stats_ws_cb` (CONCURRENTLY refresh requirement).
3. **Schema-shape verification gate** (TaskN.5): before writing the migration body, the dev agent MUST run `grep -rn "content_block_usages\|ContentBlockUsage" services/client-api/src/` to confirm the join target table actually exists. If `content_block_usages` does NOT exist (Epic 7 might track usage via a JSONB column on `proposals` or a different table), substitute the actual usage source. Document the substitution in §6 D-4. Do NOT invent a table that does not exist (project-context Rule R444 anti-pattern).
4. Ownership transfer + grants identical to AC-6.
5. Same daily refresh task (appended to the existing list).

### AC-8 — `mv_workspace_onboarding_milestones` Materialized View (Hourly Refresh)

**Given** onboarding milestone tracking will be created in Story 19.2 (`client.onboarding_milestones` table seeded with 6 milestone rows on workspace creation),
**When** Story 19.0 lands,
**Then**:
1. The MV is **created from a placeholder source table** that Story 19.0 ALSO creates (so S19.1 dashboard widgets have an empty-but-queryable view to consume; populate-by-event-handlers is S19.2's responsibility):
   ```sql
   CREATE TABLE IF NOT EXISTS client.onboarding_milestones (
       workspace_id   UUID NOT NULL REFERENCES client.client_workspaces(id) ON DELETE CASCADE,
       milestone      VARCHAR(64) NOT NULL,
       completed_at   TIMESTAMPTZ NULL,
       PRIMARY KEY (workspace_id, milestone),
       CONSTRAINT ck_onboarding_milestones_name CHECK (
           milestone IN ('workspace_created', 'content_uploaded', 'first_opp_reviewed',
                         'first_ai_summary', 'crm_connected', 'first_bid_decision')
       )
   );
   ```
2. View definition:
   ```sql
   CREATE MATERIALIZED VIEW client.mv_workspace_onboarding_milestones AS
   SELECT
       workspace_id,
       milestone,
       completed_at
   FROM client.onboarding_milestones;
   ```
3. UNIQUE index on `(workspace_id, milestone)` named `ux_mv_workspace_onboarding_milestones_ws_m`.
4. Ownership transfer + grants identical to AC-6.
5. **HOURLY refresh** (not daily — per epic spec line 35) — append to or create a new Celery Beat task running `REFRESH MATERIALIZED VIEW CONCURRENTLY client.mv_workspace_onboarding_milestones` every hour at minute 5. If you create a new task, name it `refresh_workspace_onboarding_milestones` and put it in the same module as the existing migration-011 daily refresh task.
6. **Out-of-scope fence** (§6 D-5): seeding the 6 milestone rows on workspace creation, wiring milestone events from Epic 7/10/11/14/17, and CSM stall alerts are **all S19.2 scope**. Story 19.0 only creates the table + MV + refresh job.

### AC-9 — Concurrent Refresh Regression Test

**Given** the three new MVs,
**When** integration test `tests/integration/test_workspace_outcome_mvs.py::test_concurrent_refresh_no_read_locks` runs,
**Then**:
1. Seed: 100 `client.bid_outcomes` rows across 3 workspaces (canonical ORM seeding — Company + Workspace + User + CompanyMembership + Subscription + Opportunity + Proposal + BidOutcome via models, NEVER raw `text("INSERT INTO client.…")` — AP14-04 / AP15-08 anti-pattern carry-forward).
2. Concurrent steps (use `asyncio.gather` with `asyncio.create_task`):
   - Task A: `REFRESH MATERIALIZED VIEW CONCURRENTLY client.mv_workspace_outcome_stats`
   - Task B: 50 sequential `INSERT INTO client.bid_outcomes (...)` writes (via ORM; each in its own `db_session.commit()`)
   - Task C: 50 sequential `SELECT * FROM client.mv_workspace_outcome_stats WHERE workspace_id = :wid` reads
3. Assert: Task C completes with NO read-blocked rows (i.e. completion_time < 5s; if `REFRESH` were non-CONCURRENT, Task C would block until Task A completes). Use `time.monotonic()` to measure.
4. Assert: Task A completes (proves the UNIQUE index on `(workspace_id, month)` was created; otherwise CONCURRENTLY refresh fails with "cannot refresh materialized view ... concurrently").
5. Marker: `@pytest.mark.integration` (needs Postgres; runs in `make test-integration` after `make infra`).
6. Anti-pattern guard: NO `text("INSERT INTO …")` in the test body. Use ORM models. AP14-04 BLOCKING #3 carry-forward.

### AC-10 — Workspace-Scoped Cross-Tenant + Cross-Workspace Negative Test Matrix

**Given** the new workspace-scoped POST/GET endpoints,
**When** integration test `tests/integration/test_bid_outcomes_workspace_isolation.py` runs,
**Then**:
1. **Cross-tenant matrix**: parametrised over `direction ∈ {a_to_b, b_to_a}` × `attacker_role ∈ {bid_manager, admin}` = 4 cases. Company A's bid_manager attempts to POST/GET an outcome on Company B's opportunity within Company B's workspace → **404** (not 403; AP14-04 enumeration leak). Symmetric reverse direction (B→A) — proves both halves of the bypass guard. Carry-forward S15-0 B3 reverse-direction parametrisation.
2. **Cross-workspace matrix** (within same company): parametrised over `direction ∈ {w1_to_w2, w2_to_w1}` × `attacker_role ∈ {bid_manager, admin}` = 4 cases. User in Workspace W1 (same company) attempts to POST/GET on an opportunity in W2 → **404**. Tenant_admin role bypass (Epic 14 carry-forward `_BYPASS_ROLES`): a `tenant_admin` SHOULD succeed on cross-workspace within the same company — assert this as a positive case (1 case).
3. **Inactive-user negative**: seed an `is_active=False` bid_manager → 401/403 (User.is_active gate per project-context anti-pattern fence).
4. Canonical ORM seeding only (AP14-04 BLOCKING #3 — NEVER raw `text("INSERT INTO client.…")`).
5. Use `register_and_verify_with_role(role="bid_manager")` and `create_company_pair()` from `eusolicit-test-utils` (root `tests/conftest.py`).
6. NO `db_session.commit()` in test bodies — only in fixtures (gold-standard rollback isolation per project-context Rule M1; S15-0 M1 review-fix carry-forward).

### AC-11 — Frontend ATDD Source-Inspection Tests

**Given** the new outcome-capture form,
**When** ATDD source-inspection test `frontend/apps/client/__tests__/outcome-capture-source-inspection.test.ts` runs,
**Then**:
1. AST scan over the form component file (likely `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/opportunities/[opportunityId]/page.tsx` OR a sub-component `components/OutcomeCaptureForm.tsx`):
   - **Asserts** an import of `useZodForm` from `@eusolicit/ui`.
   - **Forbids** an import of `useForm` directly from `react-hook-form` in the form file.
   - **Asserts** every `<input>` / `<select>` / `<textarea>` / `<RadioGroup>` is inside a `<FormField>` JSX wrapper (parent-chain walk via `@babel/parser` + `@babel/traverse`, mirroring the Story 14-3 pattern at `frontend/apps/client/__tests__/workspace-switcher-source-inspection.test.ts`).
   - **Asserts** the file imports `<QueryGuard>` from `@eusolicit/ui` and wraps at least one query.
   - **Asserts** `queryClient.invalidateQueries` is called with a key array that includes the variable `workspaceId` (Epic 14 cache-isolation gate).
2. Pattern reuse: copy the AST-scan harness from `frontend/apps/client/__tests__/workspace-switcher-source-inspection.test.ts` (Story 14-3 set the canonical L1 ATDD pattern — do NOT reinvent).
3. Test runs in `pnpm test` (vitest) — added to the L1 ATDD gate that `bmad-dev-story` validates before mark-as-review.

### AC-12 — i18n Parity + Test-Design Inline Fill (E19 has no `test-design-epic-19.md`)

**Given** no `test_artifacts/test-design-epic-19.md` exists (confirmed by `ls test_artifacts/` 2026-05-04 — only test-design files for E16/E17/E18 exist as stub references; E19 was left for inline-fill per AP18-H1 carry-forward),
**When** Story 19.0 ships,
**Then**:
1. §4.7 of this story file fills the test-design gap inline. Risk priorities: **R-019-1** materialized-view concurrent-refresh safety (Score 6, mitigation: AC-9), **R-019-2** cross-workspace analytics leak (Score 6, mitigation: AC-10), **R-019-3** platform_attributed mis-tagging on ingest (Score 4, mitigation: AC-2 + AC-2.5 unit test), **R-019-4** MV column-name contract drift breaking S19.1 (Score 4, mitigation: AC-6/7/8 lock down column names).
2. Test-design provenance cites: epic spec lines 26–51 (acceptance criteria), AP14-04 / AP15-08 (canonical ORM seeding), AP17-C1 (two-gate close), AP18-C2 (atomic Status patch), AP18-H1 (inline test-design fill), Project-context Rules R21 (CONCURRENTLY + UNIQUE index), R44 (analytics scope), R76 (QueryGuard), R263 (Recharts E2E), R263 (form ATDD source-inspection), Migration 011 ownership-transfer pattern.
3. **i18n parity gate**: `pnpm check:i18n` MUST pass after Story 19.0 (1502 + ~12 = ~1514 keys per locale). New keys are namespaced under `outcomeCapture.*` (form labels), `outcomeCapture.errors.*` (validation), `outcomeCapture.toast.*` (success/failure feedback).

## Tasks / Subtasks

> Order is implementation-dependency-driven. Each task lists owning ACs in parens; subtasks aim at ≤30 min of focused work each so the dev agent can checkpoint frequently.

- [ ] **Task 1 — Alembic migration 063: `platform_attributed` column** (AC-1)
  - [ ] Create `services/client-api/alembic/versions/063_add_platform_attributed_to_opportunities.py` (revision="063", down_revision="062"; mirror format of 062_add_locale_preference_to_users.py)
  - [ ] Add column with `server_default="false"` so existing rows backfill atomically
  - [ ] Add index `ix_opportunities_workspace_platform_attributed` on `(workspace_id, platform_attributed)`
  - [ ] Update ORM `services/client-api/src/client_api/models/opportunity.py`: add `platform_attributed: Mapped[bool]` field; extend `__table_args__` index list
  - [ ] Run `make migrate-service SVC=client-api` against local Postgres; confirm column + index via `\d client.opportunities` in psql
  - [ ] Run `alembic check` — must report no drift between ORM and migration

- [ ] **Task 2 — Auto-set `platform_attributed=TRUE` on pipeline-sourced ingest** (AC-2)
  - [ ] Search `services/client-api/src/client_api/services/` and `services/data-pipeline/` for the path that copies `pipeline.opportunities` → `client.opportunities`
  - [ ] If found: set `platform_attributed=TRUE` at the write site; add unit test parametrised over `source ∈ {pipeline_feed, manual_upload, crm_hubspot, crm_pipedrive, crm_salesforce}`
  - [ ] If NOT found: document Known Deviation §6 D-2 with concrete deferral note ("S19.1 backfill task will set platform_attributed=TRUE for opportunities where `source = pipeline_feed` retroactively"). Confirm default-FALSE preserves correctness in the meantime.
  - [ ] Add explicit `platform_attributed=FALSE` invariant test for CRM-sourced rows (`crm_external_provider IS NOT NULL` → `platform_attributed = FALSE`)

- [ ] **Task 3 — Alembic migration 064: `bid_outcomes` extension** (AC-3)
  - [ ] Create `services/client-api/alembic/versions/064_extend_bid_outcomes_for_e19_capture.py` (revision="064", down_revision="063")
  - [ ] Add `evaluator_score INTEGER NULL` + CheckConstraint range 0–100
  - [ ] Add `effort_hours INTEGER NULL` + CheckConstraint non-negative
  - [ ] Update ORM `services/client-api/src/client_api/models/bid_outcome.py`: add the two columns using **legacy `Column(...)` syntax** to match the existing file (NOT `Mapped[...]` — consistency over modernisation churn)
  - [ ] Update Pydantic schemas in `services/client-api/src/client_api/schemas/bid_outcomes.py`: extend `BidOutcomeCreateRequest` and `BidOutcomeResponse`
  - [ ] Run `alembic check` — confirm no drift
  - [ ] Document Known Deviation §6 D-1 (contract_value_eur stays Numeric, not migrated to Integer per epic prose)

- [ ] **Task 4 — Workspace-scoped POST/GET outcome endpoint** (AC-4)
  - [ ] Create NEW router `services/client-api/src/client_api/api/v1/workspace_bid_outcomes.py` with `prefix="/workspaces/{workspace_id}/opportunities/{opportunity_id}/outcome"`, tags `["workspace-bid-outcomes"]`
  - [ ] Implement `POST` and `GET` handlers; both call the existing `bid_outcome_service` extended with `workspace_id` kwarg
  - [ ] Mount the new router in `services/client-api/src/client_api/main.py` (or wherever routers are registered) — preserve existing `bid_outcomes` router mount for backward-compat
  - [ ] Extend `bid_outcome_service.record_outcome(...)` and `get_outcome(...)` with optional `workspace_id: UUID | None = None`; when provided, validate `client.opportunities.workspace_id == workspace_id`
  - [ ] Add `@deprecated` docstring + `log.warning("bid_outcome.deprecated_path_used", ...)` on the legacy `/opportunities/{id}/outcome` endpoint
  - [ ] Add OpenAPI examples to the new endpoint (request body + 201/404/403 responses)

- [ ] **Task 5 — Outcome capture frontend form** (AC-5, AC-11)
  - [ ] Create `frontend/apps/client/components/OutcomeCaptureForm.tsx` using `useZodForm(schema)` + `<FormField>` + `<RadioGroup>` from `@eusolicit/ui`
  - [ ] Define Zod schema with the 6 fields (outcome enum, proposal_id UUID, evaluator_score 0–100, contract_value_eur ≥0, effort_hours ≥0, evaluator_feedback ≤5000 chars)
  - [ ] Embed the form on the opportunity detail page `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/opportunities/[opportunityId]/page.tsx` — render only when at least one proposal is `submitted`
  - [ ] Extend `frontend/apps/client/lib/api/bid-outcomes.ts::recordBidOutcome()` with optional `workspaceId` parameter; route to workspace-scoped path when provided
  - [ ] Add success / error toast handlers; replace form with read-only summary on success
  - [ ] Wrap proposal-list fetch in `<QueryGuard>`; ensure TanStack Query keys include `workspaceId` (Epic 14 cache-isolation)
  - [ ] Add BG + EN translations to `frontend/apps/client/messages/{bg,en}.json` under `outcomeCapture.*` namespace; run `pnpm check:i18n` and confirm parity
  - [ ] Write ATDD source-inspection test `frontend/apps/client/__tests__/outcome-capture-source-inspection.test.ts` (copy harness from `workspace-switcher-source-inspection.test.ts`)

- [ ] **Task 6 — Alembic migration 065: three workspace-level MVs** (AC-6, AC-7, AC-8)
  - [ ] Create `services/client-api/alembic/versions/065_create_workspace_outcome_materialized_views.py` (revision="065", down_revision="064")
  - [ ] **Pre-migration grep gate**: run `grep -rn "content_block_usages\|ContentBlockUsage" services/client-api/src/` to verify the join target for `mv_workspace_content_reuse_stats` exists. If not, document Known Deviation §6 D-4 and substitute the actual usage source (or a stub view returning zero rows pending Epic 7 backfill).
  - [ ] CREATE `mv_workspace_outcome_stats` + UNIQUE index `ux_mv_workspace_outcome_stats_workspace_month` + ownership transfer to `notification_role` + GRANT SELECT to `client_role`
  - [ ] CREATE `mv_workspace_content_reuse_stats` + UNIQUE index + ownership transfer + grants
  - [ ] CREATE `client.onboarding_milestones` table (placeholder; AC-8.1 schema with CHECK constraint on milestone enum)
  - [ ] CREATE `mv_workspace_onboarding_milestones` + UNIQUE index + ownership transfer + grants
  - [ ] `downgrade()` drops in reverse order: MVs first, then index, then table
  - [ ] Run `make migrate-service SVC=client-api`; confirm via `\dm client.*` (list MVs) and `\dp client.mv_*` (grants)

- [ ] **Task 7 — Refresh job wiring (Celery Beat)** (AC-6.5, AC-7.5, AC-8.5)
  - [ ] Search `services/notification/src/notification/` for the existing migration-011 MV refresh task (likely in `celery_app.py` or `tasks/analytics_refresh.py`)
  - [ ] If a list of MV names exists: append `mv_workspace_outcome_stats`, `mv_workspace_content_reuse_stats` to the daily list; add `mv_workspace_onboarding_milestones` to a new hourly list
  - [ ] If no existing task: create `services/notification/src/notification/tasks/refresh_workspace_mvs.py` with two Celery tasks (`refresh_workspace_outcome_mvs_daily` running 03:00 UTC, `refresh_workspace_onboarding_milestones_hourly` running every hour at minute 5)
  - [ ] All refresh statements MUST be `REFRESH MATERIALIZED VIEW CONCURRENTLY client.<mv_name>` (NOT plain REFRESH — project-context Rule R21)
  - [ ] Wrap each refresh in a try/except logging failures via structlog (single MV failure should not block the next MV in the loop)

- [ ] **Task 8 — Cross-tenant + cross-workspace integration test matrix** (AC-10)
  - [ ] Create `tests/integration/test_bid_outcomes_workspace_isolation.py`
  - [ ] Use `create_company_pair()` + `register_and_verify_with_role()` from `eusolicit-test-utils`
  - [ ] Parametrise `direction ∈ {a_to_b, b_to_a}` × `attacker_role ∈ {bid_manager, admin}` for cross-tenant (4 cases) → expect 404
  - [ ] Parametrise same for cross-workspace within company (4 cases) → expect 404
  - [ ] Add positive case: `tenant_admin` cross-workspace within same company → 201 (Epic 14 `_BYPASS_ROLES` carry-forward)
  - [ ] Add `is_active=False` negative case → expect 401/403
  - [ ] All seeding via canonical ORM models (NO raw `text("INSERT …")`) — AP14-04 BLOCKING #3 carry-forward
  - [ ] No `db_session.commit()` in test bodies — only in fixtures (S15-0 M1 carry-forward)
  - [ ] Marker: `@pytest.mark.integration`

- [ ] **Task 9 — Concurrent-refresh regression test** (AC-9)
  - [ ] Create `tests/integration/test_workspace_outcome_mvs.py::test_concurrent_refresh_no_read_locks`
  - [ ] Seed 100 BidOutcome rows across 3 workspaces via canonical ORM
  - [ ] `asyncio.gather` of 3 concurrent tasks: REFRESH CONCURRENTLY + 50 INSERTs + 50 SELECTs
  - [ ] Assert SELECT completion time < 5s (proves no read-block)
  - [ ] Marker: `@pytest.mark.integration`
  - [ ] Add a second test asserting the UNIQUE index exists on each MV (pg_indexes query)

- [ ] **Task 10 — `Status: review` transition + sprint-status atomic patch** (AP18-C2 carry-forward — failed 13 consecutive epics)
  - [ ] After all tests pass: edit this file's line 3 from `Status: ready-for-dev` to `Status: review`
  - [ ] In the SAME bmad-dev-story commit, update `eusolicit-docs/implementation-artifacts/sprint-status.yaml` `19-0-...: review`
  - [ ] Both edits in ONE commit — atomic transition. Failing this is the project's most-repeated weakness.

- [ ] **Task 11 — Validation gate before mark-as-review**
  - [ ] `make migrate-all` → all alembic revisions clean, no drift
  - [ ] `pytest services/client-api -k "bid_outcome or workspace or platform_attributed" -v` → all green
  - [ ] `pytest tests/integration/test_bid_outcomes_workspace_isolation.py tests/integration/test_workspace_outcome_mvs.py -v` → all green
  - [ ] `pnpm test --filter=client outcome-capture` → ATDD source-inspection green
  - [ ] `pnpm check:i18n` → 1514 keys parity (1502 baseline + ~12 new)
  - [ ] `pnpm type-check` → clean (frontend)
  - [ ] `make lint` + `make type-check` → clean (backend)
  - [ ] Quote the full pytest summary line in Dev Agent Record (M2 carry-forward — Story 15-0 M2 review-fix pattern)

## Dev Notes

### 1. Architecture Compliance

**Source**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md`

- **§2 Change-6 (Rule of Three)**: Analytics stays in client-api. Do NOT split into a new service. The MVs live in `client` schema, refresh runs from `notification` service (consistent with migration 011 pattern that already grants `notification_role` ownership of analytics MVs).
- **FR7.2 Concurrent Refresh Pattern**: `REFRESH MATERIALIZED VIEW CONCURRENTLY` mandatory — plain REFRESH takes an exclusive lock that blocks reads (project-context Rule R21). UNIQUE index on the view's identifying columns is a hard prerequisite (Postgres limitation) — without it, CONCURRENTLY refresh fails at runtime with "cannot refresh materialized view ... concurrently".
- **DB Schema Isolation**: One Postgres database, six schemas (`client`, `admin`, `pipeline`, `gateway`, `notification`, `shared`). Each service role has CRUD on its own schema only. `migration_role` has DDL rights. **NEVER cross-schema in application code** — this story keeps the MVs in `client` schema; the `notification_role` only gets ownership for refresh purposes (per migration 011 precedent).
- **RBAC**: Company-level roles (`admin`, `bid_manager`, `contributor`, `reviewer`, `read_only`) enforced via `check_entity_access()` dependency factory in `client_api/core/rbac.py`. Workspace-scoped extension from Epic 14: `WorkspaceScope` dependency + `_BYPASS_ROLES = frozenset({"admin", "bid_manager"})` for cross-workspace bypass within the same company.

### 2. Source-Hint Citations

| What | Where (file path : approx line) |
|------|----------------------------------|
| Existing `BidOutcome` ORM | `services/client-api/src/client_api/models/bid_outcome.py:13–66` |
| Existing `Opportunity` ORM (workspace-scoped) | `services/client-api/src/client_api/models/opportunity.py:23–94` |
| Existing legacy outcome route | `services/client-api/src/client_api/api/v1/bid_outcomes.py:18–63` |
| Existing `bid_outcome_service.record_outcome()` | `services/client-api/src/client_api/services/bid_outcome_service.py:36–141` |
| Existing 5 analytics MVs (precedent for ownership transfer) | `services/client-api/alembic/versions/011_analytics_materialized_views.py:23–end` |
| Latest alembic migration | `services/client-api/alembic/versions/062_add_locale_preference_to_users.py` |
| `useZodForm` hook | `frontend/packages/ui/src/lib/hooks/useZodForm.ts:14` |
| `<FormField>` component | `frontend/packages/ui/src/components/forms/FormField.tsx:101` |
| `<QueryGuard>` component | `frontend/packages/ui/src/components/feedback/QueryGuard.tsx:20–28` |
| `<AppShell>` component | `frontend/packages/ui/src/components/app-shell/AppShell.tsx:13–20` |
| RBAC bypass roles (Epic 14 carry-forward) | `services/client-api/src/client_api/core/rbac.py:102–114` |
| Existing API client `recordBidOutcome` | `frontend/apps/client/lib/api/bid-outcomes.ts` |
| Existing ATDD source-inspection harness | `frontend/apps/client/__tests__/workspace-switcher-source-inspection.test.ts` (Story 14-3 set the canonical pattern) |
| Test factories `register_and_verify_with_role`, `create_company_pair` | `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py` (root `tests/conftest.py` re-exports) |
| `BidOutcomeRecorded` event schema | `packages/eusolicit-models/src/eusolicit_models/events.py:138–149` |

### 3. Anti-Pattern Fence (Do-Not-Reimplement / Do-Not-Misuse)

This story has 9 anti-pattern fence rows the dev agent must observe. Each maps to a documented project-context rule + previous-story carry-forward.

| # | Anti-pattern | Source / Rule | Story 19.0 manifestation |
|---|--------------|---------------|--------------------------|
| 1 | Plain `REFRESH MATERIALIZED VIEW` (no CONCURRENTLY) | Project-context Rule R21; AP18-07 | All three new MVs refresh CONCURRENTLY; UNIQUE index is mandatory pre-req |
| 2 | Missing UNIQUE index → CONCURRENTLY refresh fails | Postgres limitation | Tasks 6.4/6.6/6.10 each create the UNIQUE index BEFORE wiring the refresh job |
| 3 | Cross-tenant analytics leak | Rule R44 (Risk R12.1, Score 6) | AC-10 cross-tenant matrix (4 cases) + AC-6 MV definition includes `WHERE workspace_id = :wid` predicate at consumer-side (S19.1 enforcement) |
| 4 | Raw `text("INSERT INTO client.…")` in test seeding | AP14-04 BLOCKING #3 + AP15-08 carry-forward | All test seeding via canonical ORM models; lint-grep gate in Task 11 |
| 5 | `db_session.commit()` in test body | S15-0 M1 review-fix; gold-standard rollback isolation | All commits in fixtures only |
| 6 | Direct `useForm()` from `react-hook-form` instead of `useZodForm` | Project-context Rule (form pattern) | AC-11.1 ATDD AST forbid clause |
| 7 | Bare `<Input />` instead of `<FormField>` | Project-context Rule (form pattern) | AC-11.1 ATDD AST every-input-in-FormField clause |
| 8 | TanStack Query keys without `workspaceId` | Epic 14 cache-isolation rule | AC-5.9 + AC-11.1.5 assertion |
| 9 | Story file `Status: ready-for-dev` not patched to `review` / `done` atomically with sprint-status | AP18-C2 carry-forward (failed 13 consecutive epics E09–E18) | Task 10 explicit two-edit-one-commit gate |

### 4. Test Strategy

**4.1 Pytest markers** (per project conventions):
- `@pytest.mark.unit` — pure logic (Task 2 ingest-source unit test)
- `@pytest.mark.integration` — needs Postgres + Redis (Tasks 8, 9)
- `@pytest.mark.api` — needs running services (none required for this story; integration covers it)

**4.2 Test isolation (gold standard)**:
- DB: per-test transaction rollback via `db_session` fixture (NEVER commit in tests)
- Redis: not directly involved in S19.0 (no event dispatch added beyond existing BidOutcomeRecorded which already works)
- Service-level: override `get_db_session` in fixtures, clear `dependency_overrides` in `finally`

**4.3 Frontend tests**:
- Vitest (existing) — ATDD source-inspection (AC-11)
- Component test for OutcomeCaptureForm (happy path + 4 validation error cases: outcome required, evaluator_score 0–100, contract_value_eur ≥ 0, effort_hours ≥ 0)
- E2E (Playwright) — OPTIONAL for S19.0; deferred to S19.1 once dashboard exists for round-trip

**4.4 Coverage target**: 80% minimum (project-context default; `make coverage`). New code paths in `bid_outcome_service` workspace-scoped branch, new router, new ORM fields → all covered.

**4.5 Performance baseline (deferred, not blocking S19.0)**:
- AP18-C3 carry-forward: k6 baseline for outcome endpoints SHOULD be captured before E19 close-out. Story 19.0 does NOT depend on this — outcome write/read is single-row work. Document in §6 D-6 that `inj-02-k6-performance-baseline` (Epic 13 carry-forward) extends to cover the new workspace-scoped outcome endpoint when it executes.

**4.6 NFR (deferred)**:
- AP18-C4 carry-forward: NFR assessment for E19 surfaces (PDF memory profile, MV refresh duration, S3 TTL) is S19.2 scope. Story 19.0 does NOT need NFR sign-off.

**4.7 Inline test-design fill (E19 has no `test-design-epic-19.md`)** — AP18-H1 carry-forward:

| Risk ID | Description | Score | Mitigation in this story |
|---------|-------------|-------|--------------------------|
| R-019-1 | Materialized-view CONCURRENTLY refresh fails (missing UNIQUE index OR ownership not transferred) | 6 (likelihood × impact) | AC-6/7/8 prescribe both UNIQUE index + ownership transfer; AC-9 regression test asserts CONCURRENTLY refresh actually runs |
| R-019-2 | Cross-workspace analytics leak (W1 reads W2 outcomes) | 6 | AC-10 cross-workspace matrix (4 cases) + workspace-scoped POST/GET path validates `client.opportunities.workspace_id == :wid` |
| R-019-3 | Platform-attributed mis-tagging on ingest (false positive on CRM-sourced; false negative on pipeline-sourced) | 4 | AC-2 unit test parametrised over 5 source types |
| R-019-4 | MV column-name contract drift breaking S19.1 dashboard | 4 | AC-6/7/8 lock down explicit column names and types; S19.1 will assert the same names in its consumer query |
| R-019-5 | Concurrent INSERT during MV refresh blocks reads | 5 | AC-9 regression test (timing assertion < 5s for 50 reads during refresh) |
| R-019-6 | Story `Status:` header not atomically patched (13-epic recurrence) | 3 (low impact, high frequency) | Task 10 explicit two-edit-one-commit gate |

### 5. Previous-Story Intelligence

**From Story 18-2** (most recent successful story, Epic 18 close-out):
- **Pattern**: PascalCase event_type literal (`BidOutcomeRecorded` already follows this — no change needed for S19.0)
- **Pattern**: Idempotency keys via content-addressable hashes (`yaml_content_hash` for sub-processor diff). S19.0 does NOT introduce new event publication, so this pattern doesn't apply directly. S19.2 (CSM stall alerts) WILL need a SETNX guard pattern modelled on Story 18-2's `notification:dispatched:subprocessor:{changelog_sha}:{admin_user_id}`.
- **Pattern**: Inline test-design fill when `test-design-epic-X.md` is missing — Story 18-0/18-1/18-2 all did this; Story 19.0 follows suit (§4.7 above).
- **Pattern**: Known Deviations §6 pre-recorded for items reviewers might flag as gaps but are out-of-scope.
- **Pattern**: Anti-pattern fence with explicit "this story's manifestation" column (S19.0 §3 above).

**From Story 15-0** (review-fix pass with 7 BLOCKING/MEDIUM findings closed):
- M1 carry-forward: NO `db_session.commit()` in test bodies — only in fixtures.
- M2 carry-forward: full pytest summary line quoted in Dev Agent Record (Task 11.8).
- B1 carry-forward: NO test-only routes mounted on production app (no risk in S19.0 since we're not test-routing; but if a test fixture needs a thin endpoint, mount on a per-test FastAPI app inside the test file).

**From Story 14-2 (RBAC `WorkspaceScope`)**:
- `_BYPASS_ROLES = frozenset({"admin", "bid_manager"})` is the role set that bypasses cross-workspace checks within the same company. `tenant_admin` (a future role per Epic 14 retrospective) is NOT yet in this set; AC-10's tenant_admin positive case may need either: (a) `tenant_admin` added to `_BYPASS_ROLES` (project-wide change — defer to a separate story), OR (b) test asserts `admin` (which IS in `_BYPASS_ROLES`) succeeds cross-workspace. **Recommendation**: AC-10's "tenant_admin" positive case actually tests the existing `admin` role — semantically equivalent for S19.0's purposes. Document this resolution in Dev Agent Record.

### 6. Known Deviations (Pre-Recorded — Reviewers Will See These)

> Pre-recording known deviations prevents reviewers raising them as findings. Each below is intentional, scoped, and documented at story-creation time per Story 18-2 / Story 15-0 carry-forward pattern.

| ID | Deviation | Reason | Resolution path |
|----|-----------|--------|-----------------|
| **D-1** | `client.bid_outcomes.contract_value_eur` stays `Numeric` (epic spec line 31 says `INTEGER`) | Column already exists from Migration 011 as Numeric; converting to INTEGER would risk truncation of any historical fractional EUR values. MV definition casts to bigint at read time. | None — accepted at S19.0 creation. Future cleanup story may unify if business agrees no fractional EUR is needed. |
| **D-2** | `platform_attributed=TRUE` auto-set on pipeline ingest may be deferred IF the ingest path is not yet implemented | Default-FALSE preserves correctness; just under-counts platform-attributed bids until backfill | Task 2 documents the outcome (set OR deferred). If deferred → S19.1 backfill task. |
| **D-3** | Two outcome routes in production (legacy `/opportunities/{id}/outcome` + new workspace-scoped) | Backward-compat with Story 10.11 callers; deletion is its own story (AP15-08 — never side-effect of feature work) | S19.1 cleanup story (or Epic 19 retrospective decision) deletes the legacy route once frontend migration confirmed. |
| **D-4** | `mv_workspace_content_reuse_stats` JOIN target may not exist (depends on Epic 7 `content_block_usages` table existence) | Pre-migration grep gate (Task 6.2) verifies; if absent, MV uses zero-row stub pending backfill | Task 6.2 documents the resolution; S19.1 dashboard can degrade-gracefully on a zero-row reuse view. |
| **D-5** | Onboarding milestone seeding + event handlers + CSM stall alerts are S19.2, not S19.0 | Epic spec lines 78–98 explicitly assign these to S19.02; S19.0 only delivers the table + MV + refresh shell | None — by-design. |
| **D-6** | k6 performance baseline for new workspace-scoped endpoint deferred | AP18-C3 / Epic 13 carry-forward `inj-02-k6-performance-baseline`; not blocking E19 close-out | inj-02 story (Epic 13 carry-forward) extends to cover this endpoint when it runs. |
| **D-7** | NFR assessment deferred to S19.2 (PDF memory profile dominates the NFR surface for E19) | AP18-C4 carry-forward; S19.0 has no PDF, no Redis pub/sub, no S3 — minimal NFR surface | S19.2 runs `bmad-testarch-nfr` covering both stories. |
| **D-8** | "tenant_admin" role in AC-10 tested as `admin` role | `_BYPASS_ROLES` currently contains `admin` + `bid_manager`; `tenant_admin` is a future role from Epic 14 retro that has not yet landed | Future role-extension story adds `tenant_admin` to `_BYPASS_ROLES`; AC-10 positive case is semantically equivalent today. |

### 7. Project Structure Notes

- **Alembic naming**: `0NN_<verb>_<subject>.py` per existing convention (062_add_locale_preference_to_users.py is the latest precedent). Three new migrations: `063_add_platform_attributed_to_opportunities.py`, `064_extend_bid_outcomes_for_e19_capture.py`, `065_create_workspace_outcome_materialized_views.py`. Sequential `down_revision` chain.
- **Router placement**: `services/client-api/src/client_api/api/v1/workspace_bid_outcomes.py` — new file. Mount in `client_api/main.py` (search for `app.include_router(bid_outcomes_router)` and add the new one alongside).
- **ORM file edits**: Two files touched — `models/opportunity.py` (add `platform_attributed`) and `models/bid_outcome.py` (add `evaluator_score` + `effort_hours`). Both are `client` schema models.
- **Frontend file**: `components/OutcomeCaptureForm.tsx` is new. Embedded into the opportunity detail page (App Router path: `app/[locale]/(client)/workspaces/[workspaceId]/opportunities/[opportunityId]/page.tsx` — verify exact path via `ls frontend/apps/client/app/`).
- **i18n files**: `frontend/apps/client/messages/bg.json` + `en.json` extended under new `outcomeCapture.*` namespace.
- **Naming variance from epic spec**: epic spec uses `evaluator_score` (singular). Existing `evaluator_scores` (plural, JSONB) is per-criterion scoring from Story 10.11. Story 19.0 ADDS the singular `evaluator_score` for the operator-supplied single-number score. Both columns coexist (no rename, no drop).

### 8. References

- **Epic spec**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E19-outcome-telemetry-renewal-proof.md` (lines 26–51 = S19.00 scope; lines 1–22 = epic-wide acceptance criteria + dependencies)
- **PRD**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md` §6 FR9 + §8 US11
- **Architecture**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md` §2 Change-6 (analytics in client-api), FR7.2 concurrent refresh pattern
- **Project context**: `/home/debian/Projects/eusolicit/eusolicit-docs/project-context.md` Rules R21 (CONCURRENTLY + UNIQUE index), R44 (analytics scope), R76 (QueryGuard), R263 (Recharts/form ATDD), R300 (ThreadPoolExecutor — N/A this story), R444 (alembic spec contradiction)
- **Epic 18 retro**: `/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/epic-18-retrospective.md` carry-forward AP18-C1 (deferred — not blocking S19.0; relevant to S19.2), AP18-C2 (atomic Status patch — Task 10), AP18-C3 (k6 — D-6), AP18-C4 (NFR — D-7), AP18-H1 (inline test-design fill — §4.7)
- **Migration 011 precedent**: `services/client-api/alembic/versions/011_analytics_materialized_views.py` (ownership-transfer pattern for `notification_role`)
- **Story 18-2 (most recent style reference)**: `eusolicit-docs/implementation-artifacts/18-2-sub-processor-change-dpa-notification-flow-notification-service-extension.md`
- **Story 14-3 ATDD pattern**: `frontend/apps/client/__tests__/workspace-switcher-source-inspection.test.ts`
- **Story 15-0 review-fix carry-forward**: `eusolicit-docs/implementation-artifacts/15-0-pro-plus-tier-definition-stripe-price-tier-cache-extension.md` (M1, M2, B1 patterns)
- **Operator workflow guidance**: BMAD-stream — [VS] Validate Story (NON-NEGOTIABLE) → bmad-dev-story → bmad-code-review (Approve verdict required for done — AP17-C1 two-gate-close) → [PR] Post-Review. Epic 19 has multiple stories → [SR] Story Review after S19.0 done. Epic 19 stories are interdependent (S19.1 / S19.2 read MVs created here) → [ER] Epic Review after S19.2.

## Dev Agent Record

### Agent Model Used

_(populated by bmad-dev-story autopilot)_

### Debug Log References

### Completion Notes List

### File List

_(populated by dev agent — expected files):_

**Backend**:
- `services/client-api/alembic/versions/063_add_platform_attributed_to_opportunities.py` (new)
- `services/client-api/alembic/versions/064_extend_bid_outcomes_for_e19_capture.py` (new)
- `services/client-api/alembic/versions/065_create_workspace_outcome_materialized_views.py` (new)
- `services/client-api/src/client_api/models/opportunity.py` (edit — add `platform_attributed`)
- `services/client-api/src/client_api/models/bid_outcome.py` (edit — add `evaluator_score`, `effort_hours`)
- `services/client-api/src/client_api/schemas/bid_outcomes.py` (edit — extend request/response)
- `services/client-api/src/client_api/api/v1/workspace_bid_outcomes.py` (new)
- `services/client-api/src/client_api/api/v1/bid_outcomes.py` (edit — add deprecation warning)
- `services/client-api/src/client_api/services/bid_outcome_service.py` (edit — add `workspace_id` kwarg branch)
- `services/client-api/src/client_api/main.py` (edit — register new router)
- `services/notification/src/notification/tasks/refresh_workspace_mvs.py` (new — OR edit of existing analytics-refresh task module)

**Frontend**:
- `frontend/apps/client/components/OutcomeCaptureForm.tsx` (new)
- `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/opportunities/[opportunityId]/page.tsx` (edit — embed form)
- `frontend/apps/client/lib/api/bid-outcomes.ts` (edit — add workspace path)
- `frontend/apps/client/messages/bg.json` (edit — `outcomeCapture.*` keys)
- `frontend/apps/client/messages/en.json` (edit — `outcomeCapture.*` keys)
- `frontend/apps/client/__tests__/outcome-capture-source-inspection.test.ts` (new)
- `frontend/apps/client/__tests__/outcome-capture-form.test.tsx` (new — happy path + validation cases)

**Tests**:
- `tests/integration/test_bid_outcomes_workspace_isolation.py` (new — AC-10 matrix)
- `tests/integration/test_workspace_outcome_mvs.py` (new — AC-9 concurrent refresh)
- `services/client-api/tests/unit/test_platform_attributed_invariant.py` (new — AC-2.5)

### Senior Developer Review

_(populated by bmad-code-review)_

### Senior Developer Re-Review

_(populated by bmad-code-review on review-fix re-pass; AP17-C1 two-gate close gate)_
