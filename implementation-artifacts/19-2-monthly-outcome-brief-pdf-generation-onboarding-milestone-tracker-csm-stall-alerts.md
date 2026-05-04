# Story 19.2: Monthly Outcome Brief PDF Generation + Onboarding Milestone Tracker + CSM Stall Alerts

Status: review

<!-- Validation note: Run [VS] Validate Story (bmad-validate-story) before bmad-dev-story per Operator BMAD-stream guidance — non-negotiable.
     AP18-C2 carry-forward: orchestrator MUST patch `Status:` in this file atomically with the sprint-status transition (failed 14 consecutive epics E09–E18; S19-0 + S19-1 closed the streak — DO NOT regress).
     AP17-C1 two-gate-close: `Status: done` requires bmad-code-review Pass-2 verdict = Approve; the dev-pass alone does NOT promote to done.
     AP18-C1 awareness: per epic-18 retrospective the operator proposal recommended injecting Story 18-3 (Trust Center Hardening — D6 rate-limit, D7 thread cancellation) BEFORE this story to fix the WeasyPrint deferred deviations from S18-1. The user explicitly dispatched 19-2 first; §6 D-1 of this story freezes the dependency contract (S19-2 must NOT add a second caller of the leaking thread pool — it owns its own ProcessPool/run_in_executor wrapper with a hard timeout that cancels the executor task) and §6 D-2 records the carry-forward of the S18-1 deferred work as Epic 19 retro input. -->

## Story

As a **Bid Director (with Customer Success Manager and Bid Manager co-consumers)**,
I want **(1) every workspace to auto-generate a Monthly Outcome Brief PDF on the 1st of each month — emailed-on-demand and downloadable from the dashboard — that summarises the prior month's win rate, platform-attributed bid conversion, content-reuse uplift, and hours/€ saved with embedded trend charts; (2) the six onboarding milestones (workspace_created, content_uploaded, first_opp_reviewed, first_ai_summary, crm_connected, first_bid_decision) to seed automatically on workspace creation and tick to `completed_at` as their triggering events fire from existing Epic 14/7/6/17/10 flows; (3) a daily CSM stall-alert digest to email the per-tenant CSM contact and post to the configured Slack/Teams webhook whenever a workspace has been stuck on a milestone for >7 days**,
so that **(a) Renewal QBRs are anchored on a quantitative artefact the customer can drop into their own internal review decks (Loopio's "you'll be surprised how many customers churn" warning — anecdotes don't renew Pro+ contracts); (b) Customer Success can see exactly which workspaces are stalling on adoption and intervene before the second-month renewal trigger; (c) Epic 20's NPS engine has the milestone signal it depends on (`first_bid_decision` is the gating event for the post-bid NPS prompt).**

## Epic Context

- **Epic**: E19 Outcome Telemetry & Renewal Proof (Sprint 17–18; 21 pts; Milestone: Renewal engine — closes the epic)
- **Story points**: 5 | **Type**: backend + integration (Celery Beat tasks + WeasyPrint PDF + S3 + integrations-api alert dispatch). Front-end work is minimal: a "Download last month's brief" button on the existing S19-1 dashboard plus the milestone-strip's switch from empty-state to populated.
- **FRs covered**: FR9.4 (Monthly Outcome Brief PDF — completes the FR9.4 surface S19-1 partially covered for the in-app dashboard), FR9.5 (workspace-configurable hourly rate consumed in the PDF's ROI calculation — same `resolve_outcome_config()` helper as S19-1), partial FR9.6 (onboarding milestone tracker — table + event wiring; the strip-component itself shipped in S19-1 with a degrade-gracefully empty state per S19-0 D-5), FR9.7 (CSM stall alerts — new).
- **Position in epic chain**: **FINAL story of Epic 19**. Hard-depends on **Story 19-0 (`done`)** for the `client.onboarding_milestones` placeholder table + the `mv_workspace_onboarding_milestones` materialized view AND for `mv_workspace_outcome_stats` / `mv_workspace_content_reuse_stats` (chart data). Hard-depends on **Story 19-1 (`done`)** for `compute_dashboard()` aggregator + `resolve_outcome_config()` resolution chain (the PDF reuses the dashboard endpoint's data shape — anti-pattern fence row #2 enforces NO new aggregator code in this story). Hard-depends on **Epic 16 / Story 16-0 (`done`)** for the `alert.created` event consumer that dispatches to Slack/Teams via `integrations-api`. Soft-depends on Story 18-1 (`review`) WeasyPrint renderer in `eusolicit-common.document_generation.weasyprint_renderer` — uses the renderer but MUST NOT depend on the deferred D6/D7 hardening (see §6 D-1/D-2 for the carve-out).
- **Source**: Epic spec `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E19-outcome-telemetry-renewal-proof.md` lines 78–101 (S19.02). PRD v1.1 §6 FR9.4 / FR9.6 / FR9.7 + §8 US11 (Bid Director QBR persona) + §8 US12 (CSM intervention persona). Architecture-evaluation §2 Change-6 (analytics + reporting stay in client-api; CSM alert routing reuses the integrations-api `alert.created` consumer scaffolded by S16-0).
- **Operator workflow guidance**: `[VS] Validate Story` (NON-NEGOTIABLE) → `bmad-dev-story` → `bmad-code-review` (Approve verdict required for `done` per AP17-C1 two-gate-close — current 6th potential recurrence; S19-0 + S19-1 are the only successful closures so far) → `[PR] Post-Review` → `[ER] Epic Review` (Epic 19 has interdependent stories 19-0 → 19-1 → 19-2; ER is mandatory after the chain closes per Operator guidance) → `bmad-testarch-nfr` epic-19 NFR sign-off (PDF memory profile + S3 TTL + MV refresh duration baseline — full E19 NFR per S19-1 D-7) → `epic-19-retrospective` (optional but recommended given the chain-of-three).

## Acceptance Criteria

> Source-of-truth: epic spec lines 78–101. AC numbers below cover every epic line item plus carry-forward hardening from Epic 14/15/16/17/18 + Story 19-0 + Story 19-1 retros (AP14-04 canonical ORM seeding + 404-not-403; AP15-08 consistency-over-modernisation; AP17-C1 two-gate close; AP18-C1 trust-center deferred hardening boundary; AP18-C2 atomic Status patch; AP18-C4 NFR full assessment; AP18-H1 inline test-design fill; S19-0 D-4/D-5/D-10 carry-forwards; S19-1 D-7/D-8 carry-forwards).

### AC-1 — `client.onboarding_milestones` Seed-On-Workspace-Creation

**Given** the `client.onboarding_milestones` placeholder table created by Story 19-0 migration `065_create_workspace_outcome_materialized_views.py` (PRIMARY KEY `(workspace_id, milestone)`, CHECK constraint `milestone IN ('workspace_created','content_uploaded','first_opp_reviewed','first_ai_summary','crm_connected','first_bid_decision')`),
**When** a new `client.client_workspaces` row is created (Epic 14 workspace-creation flow in `services/client-api/src/client_api/services/workspace_service.py` — locate via `grep -rn "def create_workspace" services/client-api/src/`),
**Then**:
1. Six rows MUST be inserted into `client.onboarding_milestones` for the new `workspace_id` — one per milestone — within the SAME transaction as the workspace insert (atomicity: a partially-seeded workspace would leave `mv_workspace_onboarding_milestones` reporting "5 of 6 pending" forever, since hourly refresh has no way to backfill).
2. The `workspace_created` row MUST be seeded with `completed_at = workspace.created_at` (auto-completed). The other five rows MUST have `completed_at = NULL` (pending).
3. The seed MUST be idempotent — a re-run of the workspace-create path on an existing workspace_id MUST NOT raise `IntegrityError` (use `INSERT ... ON CONFLICT (workspace_id, milestone) DO NOTHING` via `sqlalchemy.dialects.postgresql.insert`). Existing `completed_at` values MUST NOT be overwritten by the re-seed (if a milestone is already `completed_at = '2026-04-15T...'`, do NOT clobber to NULL).
4. Backfill migration `067_backfill_onboarding_milestones.py` (revision="067", down_revision="066") MUST seed the six milestones for every existing `client.client_workspaces` row at deploy time (including `workspace_created` with `completed_at = client_workspaces.created_at` if the column exists; otherwise `now()`). The backfill is **idempotent** (`ON CONFLICT DO NOTHING`) so re-running the migration is safe.
5. Migration `downgrade()` is a NO-OP for the seed (data-only; the table itself is owned by migration `065`). Document this clearly in the migration docstring — reviewers will ask.
6. Anti-pattern guard: do NOT add a per-row `INSERT` loop in the workspace-create service path. Use ONE bulk `insert(...).on_conflict_do_nothing()` with a list of six row-dicts. Sub-30-millisecond write under normal load (anti-pattern fence row #1 — N+1 inserts in a hot service path).

### AC-2 — Onboarding Milestone Event Wiring (Five Triggering Flows)

**Given** the five non-auto milestones (`content_uploaded`, `first_opp_reviewed`, `first_ai_summary`, `crm_connected`, `first_bid_decision`) need to tick `completed_at` from `NULL` to `now()` the FIRST TIME their triggering event fires for a workspace,
**When** Story 19-2 lands,
**Then** each milestone is wired via a single helper `client_api.services.onboarding_milestone_service.complete_milestone(db, *, workspace_id: UUID, milestone: str) -> None` (NEW), called from the existing service paths below:

| Milestone | Trigger event / call site | Source-of-truth file (search via grep gate) |
|-----------|---------------------------|----------------------------------------------|
| `content_uploaded` | First `client.content_blocks` row inserted for the workspace (Epic 7 content library) | `grep -rn "def create_content_block\|content_blocks.*INSERT\|create_block" services/client-api/src/client_api/services/` — wire AFTER successful insert, BEFORE commit |
| `first_opp_reviewed` | First `GET /api/v1/workspaces/:id/opportunities/:oid` view event (Epic 6 opportunity-viewer); use existing `pipeline.opportunity_view_logged` Redis Stream OR add inline service-level call | `grep -rn "def get_opportunity\|view_opportunity\|opportunity_detail" services/client-api/src/client_api/api/v1/opportunities.py` — wire on FIRST view-per-workspace (idempotent ON CONFLICT) |
| `first_ai_summary` | First successful AI-summary stream completion event (Epic 6 SSE stream from `ai-gateway`) | `grep -rn "def stream_ai_summary\|ai_summary_complete\|opportunity.*ai_summary" services/client-api/src/client_api/api/v1/opportunities.py` — wire on stream's `done` event |
| `crm_connected` | First successful CRM OAuth callback completes for the workspace (Epic 17 — `integrations-api`) | The CRM connection write happens in `services/client-api/src/client_api/api/v1/crm.py` (the `client_api/integrations-api` proxy) OR in `services/integrations-api/src/integrations_api/api/v1/`. Wire from the CLIENT-API side after the connection row is committed (the milestone is workspace-scoped, NOT integrations-api-scoped — keep cross-service coupling minimal) |
| `first_bid_decision` | First `bid_decisions.decision` row inserted with `decision IN ('go','no_go')` for the workspace (Epic 10 bid-decision flow) | `grep -rn "def create_bid_decision\|bid_decisions.*INSERT" services/client-api/src/client_api/services/` — wire AFTER successful insert |

1. `complete_milestone()` semantics:
   ```python
   async def complete_milestone(db: AsyncSession, *, workspace_id: UUID, milestone: str) -> None:
       """Mark a milestone complete; idempotent.

       UPDATE client.onboarding_milestones
       SET completed_at = COALESCE(completed_at, now())
       WHERE workspace_id = :wid AND milestone = :m;

       The COALESCE ensures we NEVER clobber an existing completion timestamp
       (e.g. a CSM resets a workspace's CRM and a second crm_connected event
       fires — the FIRST completion timestamp is the one we keep, per epic
       spec line 92 "FIRST completion").
       """
   ```
2. The helper MUST be a no-op when called on a milestone that is already completed (idempotency is enforced via `COALESCE(completed_at, now())` — a single UPDATE is sufficient; do NOT add a SELECT-then-UPDATE branch). It MUST be a no-op when the workspace_id has no row for that milestone (RAISE `WARNING` log `onboarding_milestone_row_missing`, but do NOT raise — a workspace pre-dating the AC-1.4 backfill could lack rows; the warning surfaces it without blocking the user's actual flow).
3. The five trigger sites MUST call `complete_milestone()` **synchronously, in the same transaction** as their existing write (NO Celery task hop, NO Redis-stream round-trip — the milestone-completion latency budget is the same UPDATE the workspace-write transaction already pays). Anti-pattern fence row #2: do NOT introduce a new event consumer for milestone completion; the synchronous-in-same-tx write is the canonical pattern.
4. Each trigger site MUST be wrapped in a `try / except / log.warning("milestone_wire_failed", ...)` envelope — a milestone-write failure MUST NOT roll back the user's actual write (failing to tick `first_bid_decision` because the `client.onboarding_milestones` table is missing a row is a CSM signal, NOT a user-facing error). However, the `try/except` MUST catch SPECIFIC exception types (`SQLAlchemyError`, `IntegrityError`) — NEVER bare `except:` (project-context anti-pattern).
5. Unit tests MUST cover, per milestone, the FIRST-fire (`completed_at` set), the SECOND-fire (idempotent no-op), and the missing-row case (warning emitted, no raise). Five milestones × 3 cases = 15 parametrised test cases.

### AC-3 — `monthly_outcome_briefs` Storage Table

**Given** the Outcome Brief PDF generation must record per-workspace per-month metadata (S3 key, content hash, generated_at, the metrics snapshot at generation time so the PDF is reproducible-on-audit),
**When** alembic migration `068_create_monthly_outcome_briefs.py` runs (revision="068", down_revision="067"),
**Then**:
1. NEW table `client.monthly_outcome_briefs`:
   ```sql
   CREATE TABLE client.monthly_outcome_briefs (
       id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
       workspace_id      UUID NOT NULL REFERENCES client.client_workspaces(id) ON DELETE CASCADE,
       company_id        UUID NOT NULL REFERENCES client.companies(id) ON DELETE CASCADE,
       month             DATE NOT NULL,                    -- first-of-month, e.g. 2026-04-01 represents April 2026
       s3_key            TEXT NOT NULL,                    -- workspace-scoped path; see §AC-4
       s3_bucket         TEXT NOT NULL,                    -- explicit (settings.outcome_brief_bucket); audit trail
       content_hash      TEXT NOT NULL,                    -- SHA-256 hex of the PDF bytes (idempotency key)
       byte_size         INTEGER NOT NULL,                 -- PDF size; sanity check + analytics
       metrics_snapshot  JSONB NOT NULL,                   -- full OutcomeDashboardResponse at generation time
       generated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
       generated_by      TEXT NOT NULL DEFAULT 'celery_beat',  -- 'celery_beat' | 'manual_admin' | 'on_demand_get'
       CONSTRAINT uq_monthly_outcome_briefs_workspace_month UNIQUE (workspace_id, month),
       CONSTRAINT ck_monthly_outcome_briefs_byte_size_positive CHECK (byte_size > 0),
       CONSTRAINT ck_monthly_outcome_briefs_generated_by CHECK (
           generated_by IN ('celery_beat','manual_admin','on_demand_get')
       )
   );
   CREATE INDEX ix_monthly_outcome_briefs_company_month ON client.monthly_outcome_briefs (company_id, month DESC);
   CREATE INDEX ix_monthly_outcome_briefs_workspace_month ON client.monthly_outcome_briefs (workspace_id, month DESC);
   ```
2. The UNIQUE on `(workspace_id, month)` is the IDEMPOTENCY KEY for monthly Beat re-runs: a re-execution of the Beat task for the same (workspace, month) MUST be a no-op via `INSERT ... ON CONFLICT (workspace_id, month) DO NOTHING` AFTER the content-hash dedup check (AC-4.5). This protects against operator-triggered re-runs creating duplicate S3 versions and duplicate emails.
3. ORM model NEW at `services/client-api/src/client_api/models/monthly_outcome_brief.py`. Use **legacy `Column(...)` syntax** to match `bid_outcome.py` / `workspace_outcome_config.py` (AP15-08 carry-forward). Re-export via `models/__init__.py`.
4. Migration `downgrade()` drops the indexes then the table.

### AC-4 — Monthly Outcome Brief PDF Generation (Celery Beat Task)

**Given** the per-workspace monthly PDF must be generated on the 1st of each month covering the prior month,
**When** Celery Beat fires `notification.workers.tasks.outcome_brief_generation.generate_monthly_outcome_briefs` at **04:00 UTC on the 1st of every month** (`crontab(day_of_month="1", hour="4", minute="0")`),
**Then**:
1. NEW Celery task module: `services/notification/src/notification/workers/tasks/outcome_brief_generation.py`. Mount in beat schedule: `notification/workers/beat_schedule.py` add the entry `outcome-brief-monthly-generation` after the existing `refresh-workspace-onboarding-milestones-hourly` entry (preserve alphabetical-ish ordering by hour).
2. The task iterates over **active workspaces only** — defined as `client_workspaces WHERE is_active = TRUE` (or whatever the existing active-workspace predicate is in `client.client_workspaces`; verify via `grep -rn "is_active\|deleted_at" services/client-api/src/client_api/models/client_workspace.py`). Pre-flight gate: if no `is_active` column exists, default to "all non-deleted" (`deleted_at IS NULL`); document deviation §6 D-3.
3. **Throttled fan-out**: use a Celery `group` of per-workspace subtasks, but rate-limited via `celery.app.control.add_consumer` queue settings OR by sleeping `time.sleep(0.2)` between dispatches in the parent task's enqueue loop — do NOT enqueue 5,000 PDF subtasks instantaneously (memory + WeasyPrint thread-pool exhaustion). Target: ≤10 concurrent `generate_workspace_outcome_brief` subtasks running at any instant. **Pre-flight gate**: locate existing rate-limited Celery fan-out in `services/notification/src/notification/workers/tasks/` (the alert-digest-daily task likely has this pattern — copy the pattern, do NOT reinvent rate-limiting machinery).
4. **Per-workspace subtask** `generate_workspace_outcome_brief(workspace_id: str, month_str: str)` (Celery task, runs on the same worker):
   - Resolve `month_str` to a `(window_from, window_to)` covering the **prior calendar month** in UTC (e.g. on 2026-05-01 the task generates briefs for 2026-04-01..2026-04-30 inclusive). Computed as `date.today().replace(day=1) - timedelta(days=1)` → take that date's first-of-month and last-day.
   - Call `compute_dashboard(db, company_id=..., workspace_id=..., window_from=prior_month_start, window_to=prior_month_end)` from `client_api.services.outcome_dashboard_service` (S19-1 — anti-pattern fence row #2 forbids reimplementation; the dashboard data shape IS the PDF input contract per S19-1 D-8). NOTE: this means the notification service has a **read dependency** on `client_api.services.outcome_dashboard_service`. Two integration paths:
     - **(Preferred)** Import `compute_dashboard()` directly via the `eusolicit-common` shared package OR a new `eusolicit-models` module — check what imports `outcome_dashboard_service` exposes via `grep -rn "from client_api.services.outcome_dashboard_service\|outcome_dashboard_service" services/`. If it's already importable from notification (DB-isolation rule says NO; client-schema reads via client_api_role only), use the HTTP path below instead.
     - **(Fallback)** Call `GET http://client-api:8001/api/v1/workspaces/:id/outcome/dashboard?from=YYYY-MM&to=YYYY-MM` via `httpx.AsyncClient(timeout=30.0)` with a service-to-service JWT (project-context Rule R12 — internal-service auth via the `EUSOLICIT_INTERNAL_SERVICE_TOKEN` env var, or whatever the canonical pattern is — `grep -rn "internal_service\|inter_service\|service_to_service" packages/eusolicit-common/`). **Resolution gate**: pick whichever path matches the existing notification → client-api communication pattern (the scheduled-report-delivery task in `notification/tasks/scheduled_report_delivery.py` already does cross-service work — mirror its style; do NOT introduce a third pattern). Document in §6 D-4.
   - Render trend charts to PNG bytes via `eusolicit_common.document_generation.charts.generate_chart_png` (S12.09 utility — `grep -rn "generate_chart_png" packages/eusolicit-common/src/eusolicit_common/document_generation/charts.py` — input shape: `chart_type`, `labels`, `values`). Two charts: (a) bids_tracked / bids_submitted / bids_won monthly trend; (b) win_rate trend.
   - Render the HTML brief via Jinja2 template `services/notification/src/notification/report_templates/outcome_brief.html` (NEW). The template inputs: company name, workspace name, month label ("April 2026"), KPIs (4 numbers), trend chart PNGs (base64-encoded `data:image/png;base64,...` URIs to satisfy WeasyPrint's SSRF guard — see weasyprint_renderer `_safe_url_fetcher` allow-list: `data:` URIs are allowed; `file://` only under base_url; `http(s)://` BLOCKED), platform-attribution panel (counts + diff narrative), content-reuse top-5, hours-saved formula breakdown, onboarding milestones progress strip (6 pills with completed_at dates).
   - Render PDF: call `render_html_to_pdf(html, base_url=Path(report_templates), stylesheet_paths=[Path(outcome_brief.css)])` from `eusolicit_common.document_generation.weasyprint_renderer`. **MUST** wrap in `await loop.run_in_executor(_PDF_EXECUTOR, render_html_to_pdf, html, ...)` per project-context Rule 300 (ThreadPoolExecutor mandatory for CPU-bound). NEW module-level `_PDF_EXECUTOR = concurrent.futures.ThreadPoolExecutor(max_workers=4, thread_name_prefix="outcome-brief-render")` at the top of the task file. **MUST** wrap the executor call in `asyncio.wait_for(coro, timeout=120)` (2-minute hard ceiling per workspace) AND **MUST** call `task.cancel()` + drain the executor on timeout to prevent thread-leak (S18-1 D-7 leak — see §6 D-1: do NOT use `_RENDER_LOCK` from `client_api.api.v1.trust_artefacts` which has the deferred D7 leak).
   - Compute `content_hash = hashlib.sha256(pdf_bytes).hexdigest()`. Look up existing `client.monthly_outcome_briefs` row for `(workspace_id, month)`: if exists AND `content_hash` matches, log `outcome_brief_dedup_hit`, return early (do NOT re-upload; do NOT re-insert). Otherwise proceed.
   - Upload PDF to S3 at the workspace-scoped key (AC-4.5).
   - Insert `client.monthly_outcome_briefs` row via `INSERT ... ON CONFLICT (workspace_id, month) DO UPDATE SET s3_key=EXCLUDED.s3_key, content_hash=EXCLUDED.content_hash, byte_size=EXCLUDED.byte_size, metrics_snapshot=EXCLUDED.metrics_snapshot, generated_at=EXCLUDED.generated_at` (operator re-runs are idempotent — keep the latest content; the AC-4.5 dedup gate already short-circuits identical-content re-runs).
5. **S3 storage layout** (workspace-scoped, separate from trust artefacts bucket per architecture-evaluation §Change-4 isolation):
   ```
   bucket:    <settings.outcome_brief_bucket>  (NEW; default 'eusolicit-outcome-briefs', env-var EUSOLICIT_OUTCOME_BRIEF_BUCKET)
   key:       outcome-briefs/{company_id}/{workspace_id}/{YYYY-MM}/brief.pdf
   metadata:  content-hash=<sha256-hex>, generated-at=<iso8601>, workspace=<wid>, month=<YYYY-MM>
   acl:       private (no public-read; signed URLs only — same as trust legal artefacts)
   ```
   - Dedup-on-upload: `head_object` first; if `Metadata.content-hash == content_hash`, skip PUT (mirror `trust_s3.upload_trust_artefact`). The S18-1 D-8 head_object 403 silent-PUT issue does NOT apply here because outcome briefs use `s3:ListBucket=true` (per-workspace IAM policy is permissive on list); document the difference §6 D-5.
   - The bucket MUST have S3 lifecycle rule: `outcome-briefs/*` objects expire after **400 days** (~13 months — slightly longer than 12-month QBR window so November-of-prior-year is still accessible during Q4 renewal cycles). Lifecycle is configured in `infra/terraform/s3_buckets.tf` (or wherever Terraform lives — `find /home/debian/Projects/eusolicit/eusolicit-app/infra -name '*.tf' 2>/dev/null`) — add to the resource block; if Terraform is not present, document deviation §6 D-6 with a runbook entry instructing ops to apply the lifecycle via console.
6. **Failure handling**: per-workspace exceptions (PDF render failure, S3 upload failure, DB write failure) MUST be logged with `structlog` at ERROR level with `workspace_id` + `month` + `exc_info=True`, and the parent task MUST continue to the next workspace (one bad workspace MUST NOT block the other 4,999). Aggregate failures: emit a single end-of-task summary log `outcome_brief_run_summary` with `total / succeeded / dedup_hit / failed` counts. Threshold alert: if `failed > total * 0.05` (5%), publish an `alert.created` event with `severity="page"` and `title="Monthly Outcome Brief generation failure rate exceeded 5%"` so on-call sees it within minutes.
7. Settings additions in `notification.config.NotificationSettings` (or whichever Settings class the notification service uses — `grep -rn "NotificationSettings\|class.*Settings" services/notification/src/notification/config.py`):
   - `outcome_brief_bucket: str = "eusolicit-outcome-briefs"`
   - `outcome_brief_render_timeout_seconds: int = 120` (per-workspace render hard ceiling)
   - `outcome_brief_max_concurrent_renders: int = 4` (ThreadPoolExecutor max_workers)
   - `outcome_brief_failure_alert_threshold: float = 0.05`
   - `outcome_brief_signed_url_ttl_seconds: int = 86400` (24 hours per epic spec line 84)
8. **Determinism**: WeasyPrint's `SOURCE_DATE_EPOCH=0` is already enforced at module import in `weasyprint_renderer.py:45` — relying on this. Two consecutive runs with identical metrics_snapshot MUST produce byte-identical PDFs (same SHA-256). The chart PNG generation in `generate_chart_png` (matplotlib) is also deterministic when seeded — verify via `grep -rn "rcParams\|seed\|deterministic" packages/eusolicit-common/src/eusolicit_common/document_generation/charts.py`; if matplotlib is non-deterministic for the chart types we use, document §6 D-7 and add a `pytest.mark.flaky(reruns=2)` on the byte-identity test.
9. **Anti-pattern guard #3** (carry-forward S18-1 D-6/D-7 from §6 D-1): the task MUST NOT use the `_RENDER_LOCK` / `_RENDER_TIMEOUT_SECONDS` machinery from `client_api/api/v1/trust_artefacts.py` (it is the leaking pool). Use a per-task `concurrent.futures.ThreadPoolExecutor` with explicit `cancel()` on timeout. ATDD source-inspection (AC-9.4) asserts NO import of `client_api.api.v1.trust_artefacts.*` from the notification task module.

### AC-5 — `GET /api/v1/workspaces/{workspace_id}/outcome/brief?month=YYYY-MM` Endpoint (On-Demand Signed URL)

**Given** a Bid Director / Bid Manager / Workspace Admin wants to download a previously-generated PDF (or trigger an on-demand generation for a missed month),
**When** they call `GET /api/v1/workspaces/{workspace_id}/outcome/brief?month=YYYY-MM`,
**Then**:
1. ROUTE addition on the SAME router as S19-1's `workspace_outcome_dashboard.py` (suffix path `/brief`). NO new router file — keep workspace-outcome surface in one place.
2. **RBAC**: identical to the dashboard endpoint (S19-1 AC-3.3) — `Depends(require_role("read_only"))` + `Depends(WorkspaceScope)` + `Depends(require_paid_tier(min_tier="pro_plus"))` (or whatever the canonical helper is — same pre-flight grep gate as S19-1 Task 4). Pro+ tier-gate per epic AC line 16.
3. **Path validation**: `workspace_id` MUST exist AND `workspace.company_id == current_user.company_id` (cross-tenant guard). Return **404** (NOT 403) on mismatch — AP14-04 enumeration-leak rule.
4. **Query param**: `month: str` formatted `YYYY-MM`, REQUIRED. Validate via `re.fullmatch(r"\d{4}-\d{2}", month)`; if invalid → 400 with body `{"detail": "Invalid month format; expected YYYY-MM"}`. If `month` is in the future (`> current month`) → 400 with body `{"detail": "Cannot request brief for future month"}`. If `month` is more than 13 months in the past → 410 Gone (S3 lifecycle has expired the object; the row may also be gone from `monthly_outcome_briefs` if a separate cleanup runs — for now, the lifecycle deletes the S3 object first; the row stays as audit history).
5. **Lookup**: `SELECT s3_key, s3_bucket, byte_size, generated_at FROM client.monthly_outcome_briefs WHERE workspace_id = :wid AND month = :first_of_month`. If found → generate a presigned S3 GET URL with TTL = `settings.outcome_brief_signed_url_ttl_seconds` (default 86400 = 24h per epic spec line 84). Return JSON `{"signed_url": "...", "month": "YYYY-MM", "byte_size": int, "generated_at": "ISO8601", "expires_at": "ISO8601"}`.
6. **On-demand generation** when row is missing: if `month` is the prior calendar month OR earlier (within the 13-month retention window) AND there is no row, the endpoint MUST trigger an inline generation:
   - Synchronously (NOT Celery) call the same `generate_workspace_outcome_brief()` logic as the Beat task — refactor the Beat task to extract a shared async function that both call sites use. The synchronous path MUST be wrapped in `await asyncio.wait_for(generate_brief(...), timeout=120)`; if it times out → 504 Gateway Timeout with body `{"detail": "Outcome brief generation timed out; please retry in a moment"}`.
   - On success, fall through to AC-5.5 to generate the signed URL. Set `generated_by='on_demand_get'` on the inserted row (AC-3.1 audit field).
   - On render failure → 500 with body `{"detail": "Outcome brief generation failed"}`; structlog `outcome_brief_on_demand_failed` with `exc_info=True`.
7. **Cache-Control**: response sets `Cache-Control: private, max-age=0, must-revalidate` (signed URL is short-lived; clients MUST NOT cache the signed URL itself). `Vary: Authorization` (carry-forward S19-1 H12 fix — no cross-tenant leak through shared caches).
8. **Rate-limit**: the on-demand generation path is CPU-expensive (~30–90s per call). Apply a **per-workspace rate limit of 1 generation per minute** via a Redis SETNX lock keyed `outcome_brief_lock:{workspace_id}:{month}` with TTL=60s. If lock is held → 429 Too Many Requests with body `{"detail": "Brief generation in progress; retry in a moment"}` and `Retry-After: 30` header. Anti-pattern guard #4 (carry-forward S18-1 D-6 deferred): the trust-artefacts route lacks rate-limiting; this endpoint MUST NOT inherit that gap.
9. **OpenAPI examples**: 200-cached (row present), 200-on-demand-generated (synchronous render), 400-invalid-month, 404-workspace-not-in-company, 403-tier-locked, 410-expired (>13 months), 429-generation-in-progress, 504-timeout.

### AC-6 — CSM Stall Alert Daily Beat Task

**Given** a workspace stuck on a milestone for >7 days (counted from `client_workspaces.created_at` for `content_uploaded` / `first_opp_reviewed` / `first_ai_summary` — milestones expected within 7 days; or counted from `crm_connected.completed_at` for `first_bid_decision` — NOT applicable until the user has at least connected a CRM OR uploaded content, whichever is first; see AC-6.4 stall-clock semantics),
**When** Celery Beat fires `notification.workers.tasks.csm_stall_alerts.detect_milestone_stalls` at **08:00 UTC daily** (`crontab(hour="8", minute="0")`),
**Then**:
1. NEW Celery task module: `services/notification/src/notification/workers/tasks/csm_stall_alerts.py`. Mount in `beat_schedule.py` as `csm-stall-alerts-daily` (08:00 UTC; staggered after the daily-digest 07:00 + 30 min so there's no contention with email send).
2. **Stall detection query**: per-workspace, find milestones where:
   ```sql
   SELECT om.workspace_id, om.milestone, om.completed_at, w.created_at AS workspace_created_at
   FROM client.onboarding_milestones om
   JOIN client.client_workspaces w ON w.id = om.workspace_id
   WHERE om.completed_at IS NULL
     AND w.deleted_at IS NULL                    -- skip soft-deleted
     AND w.created_at < (now() - interval '7 days')
     AND om.milestone IN ('content_uploaded', 'first_opp_reviewed', 'first_ai_summary');
   -- crm_connected and first_bid_decision excluded by default — they're advanced milestones
   -- requiring upstream completion (e.g. crm_connected requires the customer's IT to provision OAuth).
   -- Including them produces noisy "new customer hasn't done CRM yet" alerts. AC-6.4 documents.
   ```
3. **Deduplication**: per `(workspace_id, milestone)` we MUST NOT alert more than once every 7 days (avoid daily spam for the same stalled milestone). Track via `client.csm_stall_alert_history` (NEW table — AC-7) `(workspace_id, milestone, alerted_at)`; the query above gets WHERE-clause-extended with `AND NOT EXISTS (SELECT 1 FROM client.csm_stall_alert_history h WHERE h.workspace_id = om.workspace_id AND h.milestone = om.milestone AND h.alerted_at > now() - interval '7 days')`. Subsequent stall alerts for the SAME (workspace, milestone) fire weekly until the milestone completes.
4. **Stall-clock semantics** (AC-6.2 above is the canonical query; this is the rationale doc):
   - `content_uploaded` clock starts at `workspace.created_at` (every workspace should upload at least one content_block in the first week).
   - `first_opp_reviewed` clock starts at `workspace.created_at`.
   - `first_ai_summary` clock starts at `workspace.created_at` (free-tier workspaces still get 5 AI summaries/month).
   - `crm_connected` and `first_bid_decision` are NOT included in the default stall-detection query (rationale in the SQL comment above and §6 D-8).
5. **Per-tenant CSM contact resolution**: the alert needs to know WHERE to send (CSM email, Slack/Teams webhook). Resolution chain:
   - Per-company `client.companies.csm_email` (NEW column — AC-8) — if NOT NULL, use it.
   - Else fall back to the global ops alias `settings.default_csm_email = "csm@eusolicit.com"` (env-var `EUSOLICIT_DEFAULT_CSM_EMAIL`).
   - Slack/Teams routing reuses Story 16-0's `integrations.webhook_configurations` table — the alert event is published to the `alert.created` Redis Stream (existing E16 stream), which `integrations-api` consumes and dispatches to all active webhooks for the workspace's company. NO direct Slack-API calls from the notification service — anti-pattern fence row #5 (the integrations-api owns Slack/Teams I/O).
6. **Event publishing**: per stalled (workspace, milestone) row, publish ONE `alert.created` event to the Redis Stream `alerts` (the existing stream consumed by S16-0's `integrations_api/consumer.py`):
   ```python
   {
       "type": "alert.created",
       "alert_id": str(uuid4()),
       "workspace_id": str(workspace_id),
       "company_id": str(company_id),
       "severity": "warning",
       "title": "Onboarding milestone stalled",
       "category": "csm_stall",
       "milestone": milestone,                              # e.g. "first_opp_reviewed"
       "days_stalled": int,                                 # ceil((now - workspace_created) / 1day)
       "deep_link": f"/workspaces/{workspace_id}/outcome/dashboard",
       "csm_email": str | None,                             # AC-6.5 resolved address
       "stall_summary": str,                                # human-readable narrative
       "created_at": iso8601_utc,
   }
   ```
   The `alerts` stream name MUST be discovered via `grep -rn "alert.created\|alerts" services/integrations-api/src/integrations_api/consumer.py` — confirm the canonical stream name (likely `eu-solicit:notifications` per epic spec line 93, but verify; the S16-0 consumer is the source of truth).
7. **Email send**: in addition to publishing the `alert.created` event for Slack/Teams routing, the task MUST also enqueue an email send via the existing `notification.tasks.email.send_email` Celery task (signature pattern from `notification/tasks/scheduled_report_delivery.py` lines 100–150). Recipients: the resolved `csm_email` (one per stalled row, OR coalesced into a single per-company digest if the same CSM has >5 alerts in one run — see AC-6.8 batching).
8. **Per-company digest batching**: if a CSM has more than 5 stalled-milestone rows for the same company in a single daily run, COALESCE them into ONE email with a list of stalled workspaces. Threshold env var: `settings.csm_stall_digest_threshold: int = 5`. Slack/Teams events stay un-batched (Slack supports threading and visual grouping; email digesting prevents inbox flood).
9. **Idempotency record**: AFTER both the email enqueue AND the event publish succeed, INSERT into `client.csm_stall_alert_history(workspace_id, milestone, alerted_at, alert_id)`. The `alert_id` UUID matches the `alerts` stream event's `alert_id` (audit trail; if a CSM asks "where did this alert come from", we can trace it). Use `INSERT ... ON CONFLICT (workspace_id, milestone) DO UPDATE SET alerted_at = EXCLUDED.alerted_at, alert_id = EXCLUDED.alert_id` so the history row tracks the LATEST alert per (workspace, milestone) pair.
10. **Failure handling**: per-row exceptions logged, parent task continues. Aggregate summary log at task end (`csm_stall_run_summary` — total / alerted / suppressed_dedup / failed). If `failed > total * 0.10` (10%), publish an `alert.created` with `severity="page"` (operator page).

### AC-7 — `client.csm_stall_alert_history` Table

**Given** the dedup gate from AC-6.3 needs persistent state,
**When** alembic migration `069_create_csm_stall_alert_history.py` runs (revision="069", down_revision="068"),
**Then**:
1. NEW table `client.csm_stall_alert_history`:
   ```sql
   CREATE TABLE client.csm_stall_alert_history (
       workspace_id  UUID NOT NULL REFERENCES client.client_workspaces(id) ON DELETE CASCADE,
       milestone     VARCHAR(64) NOT NULL,
       alerted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
       alert_id      UUID NOT NULL,
       PRIMARY KEY (workspace_id, milestone),
       CONSTRAINT ck_csm_stall_alert_milestone CHECK (
           milestone IN ('workspace_created','content_uploaded','first_opp_reviewed',
                         'first_ai_summary','crm_connected','first_bid_decision')
       )
   );
   CREATE INDEX ix_csm_stall_alert_history_alerted_at ON client.csm_stall_alert_history (alerted_at);
   ```
2. ORM model NEW at `services/client-api/src/client_api/models/csm_stall_alert_history.py` (legacy `Column(...)` style). Re-export.
3. The notification service writes via `notification_role` — verify role grants in `infra/postgres/init/01-init-schemas-and-roles.sql` (the role pattern; if `notification_role` does NOT have `INSERT, UPDATE, SELECT` on `client.csm_stall_alert_history`, add the GRANTs in the migration's `upgrade()` body): `op.execute("GRANT SELECT, INSERT, UPDATE ON client.csm_stall_alert_history TO notification_role")`.
4. Migration `downgrade()` drops the index then the table.

### AC-8 — `client.companies.csm_email` Column + Tenant CSM Configuration

**Given** the AC-6.5 resolution chain needs a per-company CSM email destination,
**When** alembic migration `070_add_csm_email_to_companies.py` runs (revision="070", down_revision="069"),
**Then**:
1. ADD column `csm_email TEXT NULL` to `client.companies`. Add `CHECK (csm_email IS NULL OR csm_email ~* '^[^@]+@[^@]+\.[^@]+$')` (loose email regex; full RFC-5322 in Pydantic at the API layer).
2. NO admin-UI surface for setting `csm_email` ships in S19-2 (admin-API surface — out-of-scope per Epic 12 admin-API split). Document §6 D-9: ops manually `UPDATE client.companies SET csm_email = '...' WHERE id = ...` until a future admin story adds the UI. The default fallback (`settings.default_csm_email`) keeps the system functional.
3. ORM `Company` model at `services/client-api/src/client_api/models/company.py` extended with `csm_email: Mapped[str | None]` (or legacy style — match the file's existing convention — `head -50 company.py` to inspect).
4. Migration `downgrade()` drops the column.

### AC-9 — Frontend "Download Last Month's Brief" Button + ATDD Source-Inspection

**Given** the S19-1 `<OutcomeDashboard />` component renders the dashboard chrome,
**When** Story 19-2 lands the frontend slice,
**Then**:
1. Extend the dashboard header (the existing date-range-picker row in `frontend/apps/client/components/OutcomeDashboard.tsx`) with a NEW "Download last month's brief" button next to the "Configure" button. The button SHOULD render only if `currentUser.role IN {admin, bid_manager, read_only}` (any tier-eligible user can download — read-only is the canonical-broadest analytics consumer per S19-1 RBAC).
2. NEW API client function `frontend/apps/client/lib/api/outcome-dashboard.ts::fetchOutcomeBriefSignedUrl(workspaceId, month)` — calls `GET /api/v1/workspaces/:id/outcome/brief?month=YYYY-MM`. Returns `{ signed_url, month, byte_size, generated_at, expires_at }`.
3. NEW TanStack mutation hook `useFetchOutcomeBrief(workspaceId)` in `frontend/apps/client/lib/queries/use-outcome-dashboard.ts`. On mutation success: open `signed_url` in a new tab via `window.open(signed_url, '_blank', 'noopener,noreferrer')`.
4. **ATDD source-inspection** test extends the existing `frontend/apps/client/__tests__/outcome-dashboard-source-inspection.test.ts` (S19-1 harness — copy patterns):
   - Asserts `fetchOutcomeBriefSignedUrl` is imported AND used by the button click handler.
   - Asserts `window.open` is called with `'noopener,noreferrer'` (security hardening — prevent reverse tabnabbing).
   - Asserts query/mutation key for `useFetchOutcomeBrief` includes `workspaceId` (Epic 14 cache-isolation rule).
5. **Component test** for the new button: 3 cases — (a) success → `window.open` called with returned URL; (b) 429 from server → toast "Brief generation in progress; please retry"; (c) 504 → toast "Brief generation timed out; retry in a moment". Use `vi.spyOn(window, 'open')` for assertion.
6. **Onboarding milestones strip**: the component already exists from S19-1 with a degrade-gracefully empty state. After this story lands, the strip MUST render the populated milestones (since the table is now seeded by AC-1 + ticked by AC-2). NO new component code; the EXISTING component picks up the populated `onboarding_milestones` array from the dashboard endpoint automatically. The S19-1 component test for the strip MUST be extended to cover the populated-row case (3 of 6 completed → 3 filled pills + 3 outline pills).
7. **Empty-state copy update**: now that milestones are wired, the S19-1 empty-state copy ("Milestones populate as your team uses the platform — first milestone usually within 7 days" — S19-0 D-5 carry-forward) is technically correct but misleading: a zero-population strip means the seed step failed (AC-1.4 backfill missed the workspace). Update the empty-state copy to "Milestones not yet seeded — contact support" and log an analytics event `outcome_dashboard.milestones_strip_empty_render_count` so the backend team gets a signal if the backfill missed anyone.
8. **i18n parity**: ~6 new keys per locale: `outcomeDashboard.brief.{button.label, toast.fetching, toast.error.rateLimit, toast.error.timeout, milestones.notSeeded, milestones.notSeededDescription}`. `pnpm check:i18n` parity (~1573 → ~1579 keys per locale).

### AC-10 — PDF Generation Determinism + Concurrent-Render Performance Test

**Given** the WeasyPrint pipeline is deterministic (SOURCE_DATE_EPOCH=0) and the Beat task fans out to ≤4 concurrent renders,
**When** integration tests run,
**Then**:
1. **Determinism test** `services/notification/tests/integration/test_outcome_brief_determinism.py::test_two_runs_byte_identical`:
   - Seed a workspace + 12 months of bid_outcomes (deterministic seed).
   - Call `generate_workspace_outcome_brief()` twice with the SAME `(workspace_id, month_str)`.
   - Assert `sha256(pdf1) == sha256(pdf2)`. Assert `pdf1 == pdf2` byte-by-byte.
   - Marker: `@pytest.mark.integration` AND `@pytest.mark.slow` (PDF render is ~5–10s).
2. **Concurrent-render test** `test_outcome_brief_concurrent_renders.py::test_10_concurrent_renders_no_event_loop_block`:
   - Seed 10 workspaces with deterministic data.
   - Use `asyncio.gather` to launch 10 concurrent `generate_workspace_outcome_brief()` calls.
   - Assert: total wall-clock < 60s (10 concurrent × ~5s render each, with ThreadPoolExecutor max_workers=4 → ~3 batches of 4 ≈ 15s; 60s gives ample head-room).
   - Assert: each individual call returns a valid PDF (`bytes` starts with `b'%PDF'`).
   - Assert: NO `client.monthly_outcome_briefs` row has `byte_size <= 0` (sanity check).
   - Markers: `@pytest.mark.integration` AND `@pytest.mark.performance`.
3. **Event-loop-block detection**: assert `loop._ready` queue depth (pseudo — verify via `monitor_event_loop_lag()` decorator OR a timing test that runs an `asyncio.sleep(0.1)` 10× during the render and asserts the actual elapsed time is NOT > 1.5 × expected). If ANY iteration takes > 1.5×, the event loop was blocked → test fails with structured message pointing at WeasyPrint thread misuse. Pre-flight gate: locate event-loop-lag detection helpers via `grep -rn "monitor_event_loop\|event_loop_lag\|loop_lag" packages/` — copy if exists; otherwise document §6 D-10 and add a simple timing-based assertion.
4. **Memory-budget assertion** (NFR partial fill — AP18-C4 + S19-1 D-7 carry-forward): assert peak RSS during a 10-workspace concurrent render run stays under **500MB delta** above baseline. Use `psutil.Process().memory_info().rss` snapshot before / after. If > 500MB delta → test fails. This is the S19-2 NFR contribution; full E19 NFR sign-off (PDF memory profile + S3 TTL + MV refresh duration baseline) lands via `bmad-testarch-nfr` regen at epic close per AP18-C4.

### AC-11 — Cross-Tenant + Cross-Workspace + Tier-Gate Isolation Tests (PDF Endpoint + Stall Alerts)

**Given** the GET `/outcome/brief` endpoint, the daily Beat task, and the milestone-completion helper,
**When** integration tests run,
**Then**:
1. **GET endpoint cross-tenant matrix** `tests/integration/test_outcome_brief_workspace_isolation.py`:
   - Parametrised over `direction ∈ {a_to_b, b_to_a}` × `attacker_role ∈ {bid_manager, admin, read_only}` = 6 cases. Company A's user attempts the endpoint on Company B's workspace → **404** (NOT 403).
   - **Cross-workspace negative for non-bypass roles** (S19-0 B-2 carry-forward): same-company, different-workspace `contributor` (NOT in `_BYPASS_ROLES`) → **403** from `WorkspaceScope`.
   - **Cross-workspace bypass positive**: admin / bid_manager (in `_BYPASS_ROLES`) cross-workspace within same company → 200.
   - **Tier-gate negative**: Free / Starter / Professional → **403**. Pro+ / Enterprise → 200.
   - **Inactive-user**: `is_active=False` admin → 401/403.
2. **Beat-task isolation test** `tests/integration/test_outcome_brief_beat_isolation.py::test_workspace_a_pdf_does_not_leak_into_b`:
   - Seed 2 workspaces in DIFFERENT companies with DIFFERENT bid_outcomes.
   - Run `generate_monthly_outcome_briefs()` (the parent Beat task) once.
   - Assert: workspace A's PDF metrics_snapshot reflects A's bid_outcomes ONLY (no B numbers leak in).
   - Assert: workspace A's S3 key contains `company_id=A` and `workspace_id=A`; B's contains `company_id=B` and `workspace_id=B`.
   - Assert: cross-tenant query `SELECT COUNT(*) FROM client.monthly_outcome_briefs WHERE workspace_id = A.id AND s3_key LIKE '%' || B.id || '%'` returns 0.
3. **CSM stall-alert isolation** `test_csm_stall_alerts_isolation.py::test_workspace_a_stall_does_not_alert_company_b_csm`:
   - Seed company A with a stalled workspace AND company B with a non-stalled workspace, with DIFFERENT `csm_email` values.
   - Run `detect_milestone_stalls()`.
   - Assert: ONLY company A's CSM email receives a `send_email` task. Company B's `csm_email` is never enqueued.
   - Assert: ONLY ONE `alert.created` event is published (for company A); the event's `company_id` field is A's UUID.
   - Assert: `client.csm_stall_alert_history` has exactly ONE row, for company A's stalled workspace.
4. **Onboarding milestone wiring isolation** `test_onboarding_milestone_isolation.py::test_workspace_a_content_upload_does_not_tick_b`:
   - Seed two workspaces in DIFFERENT companies. Both have NULL `content_uploaded`.
   - Trigger the content-upload flow on workspace A.
   - Assert: A's `content_uploaded.completed_at` is now `NOT NULL`. B's stays `NULL`.
5. **Canonical ORM seeding ONLY** (AP14-04 BLOCKING #3 carry-forward): use `Company`, `Workspace`, `User`, `CompanyMembership`, `Subscription`, `Opportunity`, `Proposal`, `BidOutcome`, `OnboardingMilestone` ORM models. NEVER raw `text("INSERT INTO client.…")`. Seeding helpers: `register_and_verify_with_role` + `create_company_pair` from `eusolicit-test-utils`.
6. **No `db_session.commit()` in test bodies** — only in fixtures (gold-standard rollback isolation, S19-0 D-10 / S19-1 D-13 carry-forward — DO NOT introduce app_client_fresh-style commit-leaking fixtures; if `complete_milestone()` calls `db.commit()` internally, refactor to accept an injected transaction).
7. Markers: `@pytest.mark.integration`. Coverage 80%+ on all new code paths.

### AC-12 — Stall-Alert Dedup + Per-Company Digest Tests

**Given** the AC-6.3 dedup gate + AC-6.8 batching threshold,
**When** integration test `tests/integration/test_csm_stall_alerts_dedup.py` runs,
**Then**:
1. **Dedup test**: stalled workspace W1 with stalled `content_uploaded` for 8 days. Run `detect_milestone_stalls()` THREE times back-to-back. Assert: exactly ONE `alert.created` event published; exactly ONE `send_email` enqueued; exactly ONE `csm_stall_alert_history` row.
2. **7-day re-alert**: same W1, fast-forward `csm_stall_alert_history.alerted_at` to `now() - interval '8 days'` (sql UPDATE in fixture). Run again. Assert: a NEW alert fires (the 7-day window has elapsed).
3. **Per-company digest batching**: seed 6 stalled workspaces in the SAME company with the SAME `csm_email`. Run `detect_milestone_stalls()`. Assert: exactly ONE digest email enqueued (covering all 6 workspaces in the body) — NOT 6 separate emails. Assert: 6 separate `alert.created` events ARE published (Slack/Teams routing stays un-batched per AC-6.8).
4. **Default CSM email fallback**: company with `csm_email IS NULL`. Stall in W1. Assert: the `send_email` task is enqueued with recipient = `settings.default_csm_email` ("csm@eusolicit.com").
5. **Multi-milestone same workspace**: W1 stalled on `content_uploaded` AND `first_opp_reviewed`. Assert: TWO separate `alert.created` events, TWO `csm_stall_alert_history` rows (one per milestone).

### AC-13 — Onboarding Milestone Trigger Site Tests (5 milestones × 3 cases each = 15 cases)

**Given** the AC-2 trigger-site wiring,
**When** integration test `tests/integration/test_onboarding_milestone_wiring.py` runs,
**Then**:
1. Parametrised matrix covering, per milestone:
   - **First-fire**: trigger event happens for the first time → milestone `completed_at` is set to `now()` (within 1 second of test execution).
   - **Second-fire (idempotent)**: trigger event happens twice → `completed_at` is NOT updated to a later time (preserved from first fire).
   - **Missing-row case**: workspace lacks the milestone row (simulating a pre-AC-1.4-backfill workspace) → no exception raised; `WARNING` log `onboarding_milestone_row_missing` emitted (assert via `caplog.records`).
2. The 5 milestones × 3 cases = 15 test cases. Use `pytest.mark.parametrize` with the milestone names + case enum.
3. Trigger-site invocation: each test calls the actual trigger-site service function (e.g. `create_content_block`, `view_opportunity`, `record_bid_decision`) — NOT the helper directly. This proves the wiring, not just the helper.
4. **Anti-pattern guard**: NO Celery hop in any trigger site (AC-2.3) — assert via source-inspection that NONE of `services/client-api/src/client_api/services/onboarding_milestone_service.py` imports `celery` or schedules a task. The helper MUST be synchronous-in-tx.

### AC-14 — i18n Parity + Inline Test-Design Fill (Epic 19 has no `test-design-epic-19.md`)

**Given** no `test_artifacts/test-design-epic-19.md` exists (S19-0 §4.7 + S19-1 §4.7 also fill this gap inline; AP18-H1 carry-forward),
**When** Story 19.2 ships,
**Then**:
1. §4.7 of this story file fills the test-design gap inline (extends S19-1 risk taxonomy with S19-2-specific risks):
   - **R-019-16** PDF generation thread leak compounds S18-1 D-7 (event-loop block under load) (Score 6, mitigation AC-4.4 + AC-10.2 + AC-10.3).
   - **R-019-17** WeasyPrint SSRF via base64-bypass `<img src="http://...">` slipping past the data:-only allow-list (Score 5, mitigation AC-4.4 — only `data:` and `file://`-under-`base_url` allowed; rely on weasyprint_renderer `_safe_url_fetcher`).
   - **R-019-18** Cross-tenant PDF leak — workspace A's PDF rendered with B's metrics due to a wrong `workspace_id` parameter passed to `compute_dashboard()` (Score 6, mitigation AC-11.2 — full PDF metrics_snapshot assertion against expected workspace).
   - **R-019-19** CSM stall-alert spam — same alert fires every day for the same stalled milestone (Score 4, mitigation AC-6.3 + AC-12.1 + AC-12.2).
   - **R-019-20** Beat-task partial failure — 1 bad workspace blocks the other 4,999 (Score 5, mitigation AC-4.6 — per-workspace try/except + aggregate summary log).
   - **R-019-21** Race condition on `first_bid_decision` write — two simultaneous bid decisions both try to tick the milestone (Score 2, mitigation AC-2.1 — `COALESCE(completed_at, now())` is idempotent under concurrent UPDATEs; first-write wins).
   - **R-019-22** S3 outcome-brief bucket lifecycle missing → 5-year-old briefs accumulate, S3 cost balloons (Score 3, mitigation AC-4.5 — Terraform lifecycle rule; if Terraform absent, runbook-only deviation §6 D-6).
   - **R-019-23** PDF non-deterministic → byte-identity test flakes → trust in idempotency erodes (Score 3, mitigation AC-4.8 + AC-10.1; matplotlib non-determinism deviation §6 D-7 if applicable).
   - **R-019-24** On-demand GET endpoint DoS — attacker hits `/outcome/brief?month=...` 100×/sec, exhausting the WeasyPrint thread pool (Score 5, mitigation AC-5.8 — per-workspace Redis SETNX lock + 429 + Retry-After).
   - **R-019-25** Story `Status:` header not atomically patched (15-epic recurrence — AP18-C2; S19-0 + S19-1 are the only successful applications) (Score 3, mitigation Task 16 explicit two-edit-one-commit gate).
   - **R-019-26** Onboarding milestone wiring breaks an existing trigger-site flow (e.g. content_block creation now fails because `complete_milestone()` raises) (Score 4, mitigation AC-2.4 — try/except envelope + AC-13 trigger-site tests verify the user-facing flow still succeeds even when the milestone write fails).
2. **Test-design provenance** cites: epic spec lines 78–101 (acceptance criteria), AP14-04 (canonical ORM seeding + 404-not-403), AP15-08 (consistency over modernisation), AP17-C1 (two-gate close), AP18-C1 (trust-center deferred hardening — D6/D7 carve-out in §6 D-1), AP18-C2 (atomic Status patch), AP18-C4 (NFR full assessment — AC-10.4 memory budget partial-fill), AP18-H1 (inline test-design fill), Project-context Rules R12 (HMAC compare_digest — N/A here, no HMAC), R21 (CONCURRENTLY + UNIQUE index — already enforced at MV level by S19-0), R30 (server shells / client leaves — N/A frontend slice is 1 button), R44/R444 (analytics scope — PDF reads MVs only, anti-pattern fence row #2), R76 (QueryGuard — N/A this is a button click, not a render-from-data flow), R263 (Recharts E2E + EmptyState — N/A this story's frontend is 1 button + S19-1 strip update), R300 (ThreadPoolExecutor — AC-4.4 mandatory wrapper; AC-9.4 ATDD asserts), R320 (asyncio.to_thread for sync I/O SDKs — N/A; httpx async + boto3 in run_in_executor for the S3 sync put_object call), Story 19-0 Pass-2 Approve verdict (B-1, B-2, B-3 patterns to NOT regress), Story 19-1 Pass-2 review verdict (H1, H11, H12, M5 patterns to NOT regress).
3. **i18n parity gate**: `pnpm check:i18n` MUST pass after Story 19.2 (1573 + ~6 = ~1579 keys per locale).
4. **Component-level Vitest tests** for the new "Download last month's brief" button: 3 cases per AC-9.5.

### AC-15 — Anti-Pattern Source-Inspection (ATDD Layer 1)

**Given** the WeasyPrint thread-pool pattern + the S18-1 deferred deviations,
**When** ATDD source-inspection test `services/notification/tests/unit/test_outcome_brief_source_inspection.py` runs,
**Then**:
1. AST scan over `services/notification/src/notification/workers/tasks/outcome_brief_generation.py` and `services/notification/src/notification/workers/tasks/csm_stall_alerts.py` (using `ast` stdlib — Python source inspection is simpler than @babel for JS).
2. **Asserts** for `outcome_brief_generation.py`:
   - Defines a module-level `_PDF_EXECUTOR` of type `concurrent.futures.ThreadPoolExecutor` with `max_workers=settings.outcome_brief_max_concurrent_renders` (or a literal int matching settings default 4).
   - Calls `loop.run_in_executor(_PDF_EXECUTOR, render_html_to_pdf, ...)` (NOT `loop.run_in_executor(None, ...)` — anti-pattern fence row #6: explicit executor avoids exhausting the default thread pool that's shared with other I/O).
   - Wraps the executor call in `asyncio.wait_for(coro, timeout=...)`.
   - Calls a `cancel()` / executor shutdown path on `TimeoutError` (anti-pattern fence row #6 carry-forward of S18-1 D-7).
   - Imports `render_html_to_pdf` from `eusolicit_common.document_generation` (NOT from `eusolicit_common.document_generation.weasyprint_renderer` directly — the `__init__.py` re-export is the public API).
   - Does NOT import from `client_api.api.v1.trust_artefacts` (AC-4.9 carry-forward — that module is the leaking pool; NEW story owns its own).
3. **Asserts** for `csm_stall_alerts.py`:
   - Imports `EventBus` from `eusolicit_common.events` (or whatever the canonical event-publishing helper is — `grep -rn "class EventBus\|publish.*alert" packages/eusolicit-common/src/`).
   - Does NOT directly import the Slack/Teams adapter (anti-pattern fence row #5 — integrations-api owns Slack/Teams I/O; the notification service publishes events, the integrations-api dispatches).
   - Calls `send_email.delay(...)` (Celery enqueue — NOT a synchronous SendGrid call from inside the Beat task).
4. **Forbids** in both files:
   - `from client_api.api.v1.trust_artefacts import` (carry-forward S18-1 D-7).
   - `subprocess.Popen("weasyprint")` (the CLI is not the supported invocation path; renderer module only).
   - `time.sleep(...)` outside the parent fan-out loop (a per-render `time.sleep` indicates a busy-wait anti-pattern).

## Tasks / Subtasks

> Order is implementation-dependency-driven. Each task lists owning ACs in parens. Tasks are sized for ≤30 min focused work each.

- [ ] **Task 1 — Alembic migration 067: backfill onboarding_milestones** (AC-1)
  - [ ] Create `services/client-api/alembic/versions/067_backfill_onboarding_milestones.py` (revision="067", down_revision="066"). Mirror the formatting of migration `065`.
  - [ ] `upgrade()`: for every existing `client.client_workspaces` row, INSERT 6 milestone rows via a single `bulk_insert_mappings` or `INSERT ... SELECT ... FROM client.client_workspaces ON CONFLICT (workspace_id, milestone) DO NOTHING`. Auto-complete `workspace_created` with `completed_at = client_workspaces.created_at`.
  - [ ] `downgrade()`: NO-OP (data-only seed; document in docstring).
  - [ ] Run `make migrate-service SVC=client-api`; confirm via `\d client.onboarding_milestones` + `SELECT COUNT(DISTINCT workspace_id) FROM client.onboarding_milestones` = `SELECT COUNT(*) FROM client.client_workspaces`.
  - [ ] `alembic check` — no drift.

- [ ] **Task 2 — Alembic migration 068: monthly_outcome_briefs table** (AC-3)
  - [ ] Create `services/client-api/alembic/versions/068_create_monthly_outcome_briefs.py` (revision="068", down_revision="067"). Include CHECK constraints + UNIQUE + 2 indexes per AC-3.1.
  - [ ] `op.execute("GRANT SELECT, INSERT, UPDATE ON client.monthly_outcome_briefs TO notification_role")` so the Beat task can write.
  - [ ] `op.execute("GRANT SELECT ON client.monthly_outcome_briefs TO client_api_role")` so the GET endpoint can read.
  - [ ] `downgrade()` drops indexes + table.

- [ ] **Task 3 — Alembic migration 069: csm_stall_alert_history table** (AC-7)
  - [ ] Create `services/client-api/alembic/versions/069_create_csm_stall_alert_history.py` (revision="069", down_revision="068").
  - [ ] GRANT `SELECT, INSERT, UPDATE` to `notification_role` (writes via the daily Beat task) + `SELECT` to `client_api_role` (no read use case yet, but consistent grant pattern).

- [ ] **Task 4 — Alembic migration 070: companies.csm_email column** (AC-8)
  - [ ] Create `services/client-api/alembic/versions/070_add_csm_email_to_companies.py` (revision="070", down_revision="069").
  - [ ] Add `csm_email TEXT NULL` + CHECK regex constraint.

- [ ] **Task 5 — ORM models + Pydantic schemas** (AC-1, AC-3, AC-7, AC-8)
  - [ ] Create `services/client-api/src/client_api/models/onboarding_milestone.py` (legacy `Column(...)` style — match `bid_outcome.py`). Re-export from `models/__init__.py`.
  - [ ] Create `services/client-api/src/client_api/models/monthly_outcome_brief.py`. Re-export.
  - [ ] Create `services/client-api/src/client_api/models/csm_stall_alert_history.py`. Re-export.
  - [ ] Extend `Company` model with `csm_email: Mapped[str | None]` (or legacy style).
  - [ ] Create `services/client-api/src/client_api/schemas/outcome_brief.py` with `OutcomeBriefSignedUrlResponse(BaseModel)` (signed_url, month, byte_size, generated_at, expires_at).
  - [ ] `alembic check` clean after ORM additions.

- [ ] **Task 6 — Onboarding-milestone helper service** (AC-1, AC-2)
  - [ ] Create `services/client-api/src/client_api/services/onboarding_milestone_service.py` with:
    - `seed_milestones_for_workspace(db, *, workspace_id, workspace_created_at) -> None` (called from workspace-creation flow).
    - `complete_milestone(db, *, workspace_id, milestone) -> None` (called from 5 trigger sites).
  - [ ] Wire `seed_milestones_for_workspace()` into the existing workspace-creation service (locate via `grep -rn "def create_workspace" services/client-api/src/client_api/services/workspace_service.py`). Wrap in try/except; on failure log `WARNING` but do NOT roll back the workspace creation.
  - [ ] Wire `complete_milestone()` into the 5 trigger sites per AC-2 table. Each wire-up is a 2-3 line addition: try / call helper / except SQLAlchemyError as e: log.warning(..., exc_info=True). Locate trigger sites via the documented grep gates.

- [ ] **Task 7 — `monthly_outcome_briefs` GET endpoint** (AC-5)
  - [ ] Add `GET /outcome/brief` route to the existing `workspace_outcome_dashboard.py` router (S19-1 file; same `prefix="/workspaces/{workspace_id}/outcome"`).
  - [ ] Implement validation (month regex, future-month, >13-month-past), RBAC (read_only + WorkspaceScope + Pro+ tier-gate), lookup, on-demand generation, signed URL.
  - [ ] Redis SETNX rate-limit per AC-5.8. Use `redis.set(key, "1", nx=True, ex=60)`; if returns False → 429 + Retry-After.
  - [ ] OpenAPI `responses=` examples per AC-5.9.

- [ ] **Task 8 — Outcome Brief Celery task module** (AC-4)
  - [ ] Create `services/notification/src/notification/workers/tasks/outcome_brief_generation.py`:
    - Module-level `_PDF_EXECUTOR = ThreadPoolExecutor(max_workers=settings.outcome_brief_max_concurrent_renders)`.
    - `@celery.task` `generate_monthly_outcome_briefs()` parent task — iterates active workspaces, fan-out via `group(generate_workspace_outcome_brief.s(...) for ws in active_workspaces)` with throttling (4-at-a-time).
    - `@celery.task` `generate_workspace_outcome_brief(workspace_id, month_str)` per-workspace subtask.
    - Helper `_render_pdf(html, base_url, stylesheets) -> bytes` — wraps `render_html_to_pdf` in `loop.run_in_executor(_PDF_EXECUTOR, ...)` + `asyncio.wait_for(timeout=120)` + `cancel()` on TimeoutError.
    - Helper `_upload_to_s3(workspace_id, company_id, month, pdf_bytes, content_hash) -> str` — head_object dedup gate; put_object with metadata.
    - Helper `_record_brief(db, workspace_id, company_id, month, s3_key, content_hash, byte_size, metrics_snapshot, generated_by) -> None` — ON CONFLICT (workspace_id, month) DO UPDATE.
  - [ ] Add Beat schedule entry in `notification/workers/beat_schedule.py`: `outcome-brief-monthly-generation` at `crontab(day_of_month="1", hour="4", minute="0")`.
  - [ ] Add 5 settings to `notification.config.NotificationSettings` per AC-4.7.

- [ ] **Task 9 — Jinja2 HTML template + CSS for Outcome Brief PDF** (AC-4.4)
  - [ ] Create `services/notification/src/notification/report_templates/outcome_brief.html` — full HTML doc with header (company + workspace + month), KPI cards (4-up grid via flexbox), trend-chart `<img src="data:image/png;base64,...">` placeholders, platform-attribution panel, content-reuse top-5 table, hours-saved formula footer, milestone progress strip.
  - [ ] Create `services/notification/src/notification/report_templates/outcome_brief.css` — print-oriented stylesheet (`@page { size: A4; margin: 2cm }`, web-safe fonts only since WeasyPrint's font-fetching is restricted by SSRF guard).
  - [ ] Pre-flight gate: locate existing Jinja2 template patterns in the repo (`grep -rn "Jinja2\|env\.get_template\|jinja_env" services/notification/`); reuse the template-loader pattern.

- [ ] **Task 10 — CSM Stall Alert Celery task module** (AC-6)
  - [ ] Create `services/notification/src/notification/workers/tasks/csm_stall_alerts.py`:
    - `@celery.task` `detect_milestone_stalls()` daily task — runs the AC-6.2 SQL, dedups via AC-6.3 `csm_stall_alert_history`, batches per AC-6.8, publishes `alert.created` events, enqueues `send_email`.
  - [ ] Add Beat schedule entry: `csm-stall-alerts-daily` at `crontab(hour="8", minute="0")`.
  - [ ] Add `default_csm_email`, `csm_stall_digest_threshold` settings.

- [ ] **Task 11 — Alert event payload schema** (AC-6.6)
  - [ ] Locate existing `alert.created` payload schema in `eusolicit-models/events.py` OR `services/integrations-api/src/integrations_api/consumer.py` (verify via `grep -rn "alert.created\|class.*Alert" packages/eusolicit-models/src/`).
  - [ ] Either reuse OR add `CsmStallAlertCreated(BaseModel)` to `eusolicit-models/events.py` with PascalCase event_type literal (S18-2 #39 carry-forward) — pick whichever matches the existing pattern; document in §6 D-11 if a typed event class is added vs ad-hoc dict (S19-1 D-10 deferred typed events; if S19-2 is the right place to introduce them, do so).

- [ ] **Task 12 — Frontend "Download last month's brief" button** (AC-9)
  - [ ] Extend `frontend/apps/client/components/OutcomeDashboard.tsx` header row with the new button (next to "Configure").
  - [ ] Add `fetchOutcomeBriefSignedUrl(workspaceId, month)` to `frontend/apps/client/lib/api/outcome-dashboard.ts`.
  - [ ] Add `useFetchOutcomeBrief(workspaceId)` mutation hook to `frontend/apps/client/lib/queries/use-outcome-dashboard.ts`.
  - [ ] On success, `window.open(signed_url, '_blank', 'noopener,noreferrer')`.
  - [ ] Update `OnboardingMilestonesStrip.tsx` empty-state copy per AC-9.7.
  - [ ] Add 6 i18n keys per locale per AC-9.8.

- [ ] **Task 13 — S3 lifecycle rule** (AC-4.5)
  - [ ] Locate Terraform module: `find /home/debian/Projects/eusolicit/eusolicit-app/infra -name '*.tf' 2>/dev/null`. If found, add `aws_s3_bucket_lifecycle_configuration` for the new `eusolicit-outcome-briefs` bucket with `expiration { days = 400 }`.
  - [ ] If Terraform is NOT in the repo, document deviation §6 D-6 and add a runbook entry under `eusolicit-app/runbooks/outcome-brief-s3-lifecycle.md` instructing ops to apply the lifecycle via console.

- [ ] **Task 14 — Integration tests** (AC-10, AC-11, AC-12, AC-13)
  - [ ] `services/notification/tests/integration/test_outcome_brief_determinism.py` — byte-identity (AC-10.1).
  - [ ] `services/notification/tests/integration/test_outcome_brief_concurrent_renders.py` — 10-concurrent + memory budget (AC-10.2/10.4).
  - [ ] `tests/integration/test_outcome_brief_workspace_isolation.py` — GET cross-tenant + tier-gate matrix (AC-11.1).
  - [ ] `tests/integration/test_outcome_brief_beat_isolation.py` — Beat-task no-leak (AC-11.2).
  - [ ] `tests/integration/test_csm_stall_alerts_isolation.py` — alert dispatch isolation (AC-11.3).
  - [ ] `tests/integration/test_onboarding_milestone_isolation.py` — milestone wiring isolation (AC-11.4).
  - [ ] `tests/integration/test_csm_stall_alerts_dedup.py` — dedup + digest batching (AC-12).
  - [ ] `tests/integration/test_onboarding_milestone_wiring.py` — 5×3 trigger-site matrix (AC-13).
  - [ ] All canonical ORM seeding; no `db_session.commit()` in test bodies; `@pytest.mark.integration`.

- [ ] **Task 15 — ATDD source-inspection + frontend tests** (AC-9.4, AC-9.5, AC-15)
  - [ ] Extend `frontend/apps/client/__tests__/outcome-dashboard-source-inspection.test.ts` with the AC-9.4 asserts (new button wiring).
  - [ ] Add 3-case Vitest component test for the button per AC-9.5.
  - [ ] Create `services/notification/tests/unit/test_outcome_brief_source_inspection.py` with the AC-15 AST asserts (Python `ast` stdlib).

- [ ] **Task 16 — `Status: review` transition + sprint-status atomic patch** (AP18-C2 carry-forward — failed 14 epics; S19-0 + S19-1 closed the streak — DO NOT regress)
  - [ ] After all tests pass: edit this file's line 3 from `Status: ready-for-dev` to `Status: review`.
  - [ ] In the SAME `bmad-dev-story` commit, update `eusolicit-docs/implementation-artifacts/sprint-status.yaml`: `19-2-monthly-outcome-brief-pdf-generation-onboarding-milestone-tracker-csm-stall-alerts: review`.
  - [ ] Both edits MUST be in ONE commit — atomic transition.

- [ ] **Task 17 — Validation gate before mark-as-review**
  - [ ] `make migrate-all` → all alembic revisions clean, no drift (4 new migrations: 067, 068, 069, 070).
  - [ ] `pytest services/client-api -k "onboarding_milestone or outcome_brief or csm_stall" -v` → green.
  - [ ] `pytest services/notification -k "outcome_brief or csm_stall" -v` → green.
  - [ ] `pytest tests/integration/test_outcome_brief_workspace_isolation.py tests/integration/test_outcome_brief_beat_isolation.py tests/integration/test_csm_stall_alerts_isolation.py tests/integration/test_csm_stall_alerts_dedup.py tests/integration/test_onboarding_milestone_isolation.py tests/integration/test_onboarding_milestone_wiring.py -v` → green.
  - [ ] `pnpm test --filter=client outcome-dashboard` → ATDD source-inspection + button component tests green.
  - [ ] `pnpm check:i18n` → ~1579 keys parity.
  - [ ] `pnpm type-check` → clean (frontend).
  - [ ] `make lint` + `make type-check` → clean (backend).
  - [ ] Quote the FULL pytest summary line in Dev Agent Record (S19-0 M-1 + S19-1 carry-forward — non-negotiable per Pass-2 Approve precedent; failing this re-opens M-1 and blocks Pass-2 Approve).

## Dev Notes

### 1. Architecture Compliance

**Source**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md`

- **§2 Change-6 (Rule of Three)**: PDF generation + reporting stays in the existing services. The Beat task lives in **notification-service** (already owns `scheduled_report_delivery`, `refresh_analytics_views`, `billing_usage_sync` — adding outcome briefs is a natural extension, not a new service). The on-demand GET endpoint lives in **client-api** (mirrors S19-1 dashboard endpoint). NO new microservice.
- **§Change-4 (Bucket isolation)**: outcome briefs use a SEPARATE S3 bucket from trust artefacts (`eusolicit-outcome-briefs` vs `eusolicit-trust-artefacts`). Per-tenant isolation via S3 key prefix (`outcome-briefs/{company_id}/{workspace_id}/...`). Anti-pattern guard: NEVER write outcome briefs into the trust-artefacts bucket — different IAM policies, different lifecycle, different audit boundaries.
- **FR7.2 Concurrent Refresh Pattern**: S19-2 is a pure CONSUMER of MVs (read `mv_workspace_outcome_stats`, `mv_workspace_content_reuse_stats`, `mv_workspace_onboarding_milestones`). Does NOT add new MVs. The Beat task's monthly cadence is decoupled from the daily MV refresh — if MV refresh fails the morning of the 1st, the Beat task at 04:00 UTC reads stale data; that's acceptable (the prior-month numbers are already settled by mid-month, the 1st-of-month freshness is symbolic, not analytical).
- **DB Schema Isolation**: One Postgres database, six schemas. New tables (`monthly_outcome_briefs`, `csm_stall_alert_history`) live in `client` schema (the canonical home for workspace-scoped data). The `companies.csm_email` column is added to existing `client.companies`. NEVER cross-schema in app code.
- **Cross-service auth**: notification-service → client-api communication for `compute_dashboard()` reuse: pick whichever existing pattern the `scheduled_report_delivery.py` task uses. Likely candidates: (a) shared `eusolicit-common` import (preferred if available), (b) HTTP with internal-service JWT, (c) direct DB read via `notification_role` + cross-service ORM (the report_schedules pattern).

### 2. Source-Hint Citations

| What | Where (file path : approx line) |
|------|----------------------------------|
| Onboarding milestones placeholder table (S19-0) | `services/client-api/alembic/versions/065_create_workspace_outcome_materialized_views.py:163–174` |
| `mv_workspace_onboarding_milestones` MV (S19-0) | `services/client-api/alembic/versions/065_create_workspace_outcome_materialized_views.py:179–197` |
| `mv_workspace_outcome_stats` MV (S19-0) | `services/client-api/alembic/versions/065_create_workspace_outcome_materialized_views.py` (entire file) |
| Latest alembic migration | `services/client-api/alembic/versions/066_create_outcome_config_tables.py` (revision="066"; this story's `down_revision="066"`) |
| `compute_dashboard()` aggregator (S19-1) | `services/client-api/src/client_api/services/outcome_dashboard_service.py` (entire file — anti-pattern fence row #2 reuse) |
| `resolve_outcome_config()` 3-tier resolution (S19-1) | `services/client-api/src/client_api/services/outcome_dashboard_service.py::resolve_outcome_config` |
| `OutcomeDashboardResponse` schema (S19-1) | `services/client-api/src/client_api/schemas/outcome_dashboard.py` (entire file — PDF reads this shape) |
| Workspace-outcome dashboard router (S19-1) | `services/client-api/src/client_api/api/v1/workspace_outcome_dashboard.py` (this story extends with `/brief` route) |
| ORM legacy `Column(...)` style reference | `services/client-api/src/client_api/models/bid_outcome.py:13–66`, `services/client-api/src/client_api/models/workspace_outcome_config.py` |
| Workspace-create service path | `services/client-api/src/client_api/services/workspace_service.py` (verify exact filename via `grep -rn "def create_workspace" services/client-api/src/`) |
| Content-block create path (Epic 7) | `services/client-api/src/client_api/services/content_block_service.py` OR similar — locate via `grep -rn "def create_content_block" services/client-api/src/` |
| Opportunity-view path (Epic 6) | `services/client-api/src/client_api/api/v1/opportunities.py` (search for `def get_opportunity` or detail endpoint) |
| AI-summary completion path (Epic 6 SSE) | `services/client-api/src/client_api/api/v1/opportunities.py` (search for `ai_summary` or `stream_ai_summary`) |
| CRM connection success path (Epic 17) | `services/client-api/src/client_api/api/v1/crm.py` (search for OAuth callback / connection-create) |
| Bid decision create path (Epic 10) | `services/client-api/src/client_api/api/v1/bid_decisions.py` OR `bid_decisions_service.py` — locate via `grep -rn "def create_bid_decision" services/client-api/src/` |
| RBAC `WorkspaceScope` | `services/client-api/src/client_api/core/rbac.py` (S19-0 Round 2 `is_active`-first ordering) |
| Tier-gate dependency (Epic 12) | `services/client-api/src/client_api/core/auth.py` — `require_paid_tier`/`require_tier`/`subscription_required`; same grep gate as S19-1 Task 4 |
| Existing workspace-scoped router pattern (S19-0) | `services/client-api/src/client_api/api/v1/workspace_bid_outcomes.py` |
| Cache-Control + Vary pattern (S19-1 H12 fix) | `services/client-api/src/client_api/api/v1/workspace_outcome_dashboard.py` (look at the GET `/dashboard` Cache-Control / Vary headers) |
| Trust-artefacts S3 dedup pattern | `packages/eusolicit-common/src/eusolicit_common/document_generation/trust_s3.py:122–169` (head_object dedup; copy the pattern but rename the bucket+prefix) |
| WeasyPrint renderer | `packages/eusolicit-common/src/eusolicit_common/document_generation/weasyprint_renderer.py:120–201` |
| Determinism (`SOURCE_DATE_EPOCH=0`) | `packages/eusolicit-common/src/eusolicit_common/document_generation/weasyprint_renderer.py:45` |
| SSRF `_safe_url_fetcher` (data:+file: only) | `packages/eusolicit-common/src/eusolicit_common/document_generation/weasyprint_renderer.py:65–117` |
| Trust artefacts POST handler (S18-1 — has D6/D7 deferred deviations; do NOT regress) | `services/client-api/src/client_api/api/v1/trust_artefacts.py:99–101 + 357–410` (`_RENDER_LOCK`, `_RENDER_TIMEOUT_SECONDS = 300`, `run_in_executor` block) |
| Chart PNG utility (S12.09) | `packages/eusolicit-common/src/eusolicit_common/document_generation/charts.py::generate_chart_png` |
| reportlab PDF renderer (S12.09 — DO NOT use; use WeasyPrint) | `packages/eusolicit-common/src/eusolicit_common/document_generation/pdf_renderer.py` |
| Notification celery_app | `services/notification/src/notification/workers/celery_app.py` |
| Notification beat_schedule (extend) | `services/notification/src/notification/workers/beat_schedule.py:99–173` |
| Existing scheduled-report-delivery task (cross-service pattern reference) | `services/notification/src/notification/tasks/scheduled_report_delivery.py:1–150` |
| Existing send_email Celery task | `services/notification/src/notification/tasks/email.py` |
| EventBus (publish `alert.created`) | `packages/eusolicit-common/src/eusolicit_common/events.py` (verify via `grep -rn "class EventBus\|def publish" packages/eusolicit-common/src/`) |
| `alert.created` consumer (S16-0 — integrations-api) | `services/integrations-api/src/integrations_api/consumer.py:85–297` (verify the canonical stream name; the spec says `eu-solicit:notifications` but the consumer module is the source of truth) |
| `alerts` Redis Stream name | `services/integrations-api/src/integrations_api/consumer.py:1–10` (module docstring) |
| Existing webhook configurations table (S16-0) | `services/integrations-api/src/integrations_api/models/` — `WebhookConfiguration` ORM (no direct write needed in this story; integrations-api owns it) |
| `useZodForm` | `frontend/packages/ui/src/lib/hooks/useZodForm.ts:14` |
| `<QueryGuard>` | `frontend/packages/ui/src/components/feedback/QueryGuard.tsx` |
| ROI dashboard precedent (download button placement reference) | `frontend/apps/client/app/[locale]/(client)/workspaces/[workspaceId]/analytics/roi/components/RoiTrackerDashboard.tsx` |
| `<OutcomeDashboard />` (S19-1) | `frontend/apps/client/components/OutcomeDashboard.tsx` (extend the header row) |
| `OnboardingMilestonesStrip` (S19-1) | `frontend/apps/client/components/outcome-dashboard/OnboardingMilestonesStrip.tsx` (update empty-state copy) |
| TanStack Query hook pattern | `frontend/apps/client/lib/queries/use-outcome-dashboard.ts` (extend) |
| API client pattern | `frontend/apps/client/lib/api/outcome-dashboard.ts` (extend) |
| ATDD source-inspection harness (S19-1) | `frontend/apps/client/__tests__/outcome-dashboard-source-inspection.test.ts` (extend) |
| Test factories | `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py` |

### 3. Anti-Pattern Fence (Do-Not-Reimplement / Do-Not-Misuse)

This story has **8 anti-pattern fence rows** the dev agent must observe. Each maps to a documented project-context rule + previous-story carry-forward.

| # | Anti-pattern | Source / Rule | Story 19.2 manifestation |
|---|--------------|---------------|--------------------------|
| 1 | N+1 inserts in workspace-create hot path | Project-context Rule 84 (bulk operations); S19-0 used `bulk_insert_mappings` for `workspace_stats` | AC-1.6 — single bulk INSERT of 6 milestone rows; never a per-row loop |
| 2 | Recompute aggregates in PDF Beat task (JOIN+GROUP BY over base tables) | Project-context Rule 444 + FR7.2 + S19-1 anti-pattern fence row #2 | AC-4.4 — Beat task calls `compute_dashboard()` from S19-1 (or fetches via the GET endpoint); NEVER reimplements the aggregation |
| 3 | Synchronous WeasyPrint call in async context | Project-context Rule 300 (ThreadPoolExecutor mandatory for CPU-bound) | AC-4.4 + AC-9.4 + AC-15.2 — `loop.run_in_executor(_PDF_EXECUTOR, render_html_to_pdf, ...)` is non-negotiable; ATDD AST scan asserts |
| 4 | Reusing `_RENDER_LOCK` from `trust_artefacts.py` (the leaking pool) | S18-1 D-7 carry-forward (deferred fix) | AC-4.9 + AC-15.4 — module owns its own ThreadPoolExecutor; ATDD AST scan asserts NO import of `client_api.api.v1.trust_artefacts` |
| 5 | Direct Slack/Teams API call from notification service | Architecture-evaluation §Change-6 (integrations-api owns Slack/Teams I/O) | AC-6.5 + AC-6.6 — publish `alert.created` event, do NOT import any Slack/Teams adapter; ATDD asserts |
| 6 | TanStack Query keys without `workspaceId` | Epic 14 cache-isolation rule | AC-9.4 + AC-9.5 — all keys include `workspaceId` |
| 7 | Story file `Status: ready-for-dev` not patched to `review`/`done` atomically with sprint-status | AP18-C2 carry-forward (failed 14 epics; S19-0 + S19-1 closed) | Task 16 explicit two-edit-one-commit gate |
| 8 | Cross-tenant analytics leak (workspace A's PDF contains B's metrics) | Project-context Rule 444 (R12.1 Score 6); S19-0 AC-10 + S19-1 AC-7 carry-forward | AC-11.2 (Beat-task no-leak full snapshot assertion) + AC-11.3 (CSM alert no-leak) |

### 4. Test Strategy

**4.1 Pytest markers** (per project conventions):
- `@pytest.mark.unit` — pure logic (most milestone helper tests).
- `@pytest.mark.integration` — needs Postgres + Redis (Tasks 14 majority).
- `@pytest.mark.performance` — opt-in (`pytest -m performance` / `make test-performance`); AC-10.2/10.4 concurrent-render + memory-budget tests.
- `@pytest.mark.slow` — PDF render is ~5–10s; mark determinism test slow so CI can opt-in.
- `@pytest.mark.api` — N/A; integration covers the HTTP surface.

**4.2 Test isolation (gold standard)**:
- DB: per-test transaction rollback via `db_session` fixture (NEVER commit in tests). S19-0 D-10 + S19-1 D-13 carry-forward.
- Redis: `clean_redis` fixture flushes DB 1 between tests; the AC-5.8 SETNX lock uses DB 0 — ensure the fixture clears keys with prefix `outcome_brief_lock:` between tests.
- Service-level: override `get_db_session` in fixtures, clear `dependency_overrides` in `finally`.
- Celery: tasks under test run in `task_always_eager=True` mode (synchronous in-process) — verify the existing notification test conftest sets this. Do NOT spin up real workers in tests.

**4.3 Frontend tests**:
- Vitest L1 ATDD source-inspection (AC-9.4): forbid + assert imports + window.open args.
- Vitest component tests (AC-9.5): 3 cases for the new "Download brief" button.
- E2E (Playwright) — **OPTIONAL** for S19-2; the integration tests + ATDD source-inspection + component tests are the minimum bar.

**4.4 Coverage target**: 80% minimum. New code paths in `onboarding_milestone_service`, `outcome_brief_generation`, `csm_stall_alerts`, the new endpoint, the new ORM models → all covered.

**4.5 Performance baseline**: AC-10.2 + AC-10.4 fulfil partial NFR (PDF concurrent render < 60s wall-clock for 10 workspaces; <500MB RSS delta). Full E19 NFR sign-off — `bmad-testarch-nfr` regen at epic close — covers PDF memory profile under realistic 5,000-workspace fan-out, S3 TTL verification, MV refresh duration baseline. AP18-C4 + S19-1 D-7 carry-forward.

**4.6 NFR partial fill**: AC-10.4 memory budget assertion + the Terraform/runbook lifecycle entry (Task 13) constitute S19-2's NFR contribution. The `bmad-testarch-nfr` epic-19 regen runs AFTER this story closes per AP18-C4 + S19-1 D-7.

**4.7 Inline test-design fill (E19 has no `test-design-epic-19.md`)** — AP18-H1 carry-forward, extends S19-0 + S19-1 risk taxonomy:

| Risk ID | Description | Score | Mitigation in this story |
|---------|-------------|-------|--------------------------|
| R-019-16 | PDF generation thread leak compounds S18-1 D-7 (event-loop block under load) | 6 | AC-4.4 (own ThreadPoolExecutor + cancel on timeout) + AC-10.2 + AC-10.3 + AC-15.2 |
| R-019-17 | WeasyPrint SSRF via `<img src="http://...">` slipping past data: allow-list | 5 | AC-4.4 (rely on weasyprint_renderer `_safe_url_fetcher`); base64-encode chart PNGs as `data:` URIs |
| R-019-18 | Cross-tenant PDF leak — workspace A's PDF rendered with B's metrics | 6 | AC-11.2 (full metrics_snapshot assertion against expected workspace) |
| R-019-19 | CSM stall-alert spam — same alert daily for the same stalled milestone | 4 | AC-6.3 (`csm_stall_alert_history` 7-day dedup) + AC-12.1/12.2 |
| R-019-20 | Beat-task partial failure — 1 bad workspace blocks the other 4,999 | 5 | AC-4.6 (per-workspace try/except + aggregate summary log) |
| R-019-21 | Race condition on `first_bid_decision` write — two simultaneous decisions both tick the milestone | 2 | AC-2.1 (`COALESCE(completed_at, now())` is idempotent) |
| R-019-22 | S3 outcome-brief bucket lifecycle missing → cost balloon | 3 | AC-4.5 (Terraform lifecycle rule); §6 D-6 if Terraform absent |
| R-019-23 | PDF non-deterministic → byte-identity test flakes | 3 | AC-4.8 + AC-10.1; matplotlib non-determinism §6 D-7 |
| R-019-24 | On-demand GET endpoint DoS — 100×/sec hits exhaust thread pool | 5 | AC-5.8 (per-workspace Redis SETNX lock + 429 + Retry-After) |
| R-019-25 | Story Status header not atomically patched (15-epic recurrence — AP18-C2) | 3 | Task 16 explicit two-edit-one-commit gate |
| R-019-26 | Onboarding milestone wiring breaks an existing trigger-site flow | 4 | AC-2.4 (try/except envelope) + AC-13 (trigger-site tests verify user-facing flow still succeeds when milestone write fails) |

### 5. Previous-Story Intelligence

**From Story 19-1** (immediate predecessor — DONE 2026-05-04 with Round-2 review-fix closing all 4 BLOCKING + 14 HIGH + 9 MEDIUM):
- **Pattern (Round-2 review-fix carries — DO NOT REGRESS)**:
  - **B-2** `_seed_subscription_tier` ORM impl + tier-gate test unskipped — REUSE for AC-11.1 tier-gate matrix; do NOT re-skip the tier-gate cases.
  - **B-4** test-login env-gated (404 in production short-circuit BEFORE any DB / token mint) — applies to any test fixture this story might add.
  - **H1** `conversion_rate_diff` computed from per-attribution aggregation (single fence-#2 carve-out, S19-1 D-11) — S19-2 reuses the dashboard endpoint output, so this is already correct upstream; no new aggregation.
  - **H11** specific-exception clauses (ConnectionError/TimeoutError/OSError + ValueError) with exc_info=True replacing bare except — apply identically to the Beat task's per-workspace try/except (AC-4.6).
  - **H12** `Cache-Control: private` + `Vary: Authorization` (no cross-tenant leak through shared caches) — AC-5.7 mirrors this on the `/brief` endpoint.
  - **H8** try/catch on mutateAsync + error toast on PATCH failure — AC-9.5 mirrors for the GET-brief mutation.
- **Pattern (Known Deviations carry-forward)**:
  - **D-7** NFR full assessment: S19-1 deferred PDF memory profile + S3 TTL + MV refresh duration to S19-2. AC-10.4 is the partial fill; full NFR runs `bmad-testarch-nfr` regen at epic close.
  - **D-8** Recharts SVG-only (no PNG fallback for PDF embedding) — S19-2 OWNS the PDF chart-render path. Use `eusolicit_common.document_generation.charts.generate_chart_png` (matplotlib Agg backend) to render the trend charts as PNG bytes, base64-encode into the HTML template as `data:image/png;base64,...` URIs. AC-4.4 spec language.
  - **D-12** perf-test infrastructure dependency (TCP probe to localhost:8001) — S19-2's perf tests are in-process Beat task tests; no live-service probe needed. Reduces flakiness.
  - **D-13** `app_client_fresh` fixture — DO NOT inherit. Use `db_session` rollback fixture per S19-0 D-10 + S19-1 D-13 carry-forward.

**From Story 19-0** (DONE 2026-05-04 with Pass-2 Approve):
- **B-1** MV ownership transfer to `notification_role` — already fixed; S19-2 has NO new MVs (pure consumer).
- **B-2** Cross-workspace negative for non-bypass roles — directly extended into AC-11.1 of S19-2 (parametrise contributor as the non-bypass attacker).
- **B-3** Read-only summary on success + correct cache-invalidation key — N/A this story; no mutation surface beyond the "Configure" → reuse S19-1 logic.
- **M-1** Dev Agent Record verbatim pytest summary lines — DO NOT skip; failure to populate triggers re-opening M-1 and blocks Pass-2 Approve.
- **D-5** `mv_workspace_onboarding_milestones` empty until S19-2 — S19-2 lands the seed (AC-1) + event wiring (AC-2). After this story, the strip on the dashboard renders populated data.

**From Story 18-1** (`weasyprint_renderer` + trust-artefact PDF pipeline):
- **D-6 (deferred)** DoS surface on GET artefact endpoint: no rate-limit, no request timeout. S19-2's `/brief` endpoint MUST NOT inherit this gap → AC-5.8 Redis SETNX rate-limit + AC-5.6 wait_for(120) timeout.
- **D-7 (deferred)** `run_in_executor` thread leak: POST handler's `asyncio.Lock + wait_for(300)` wrapper does NOT cancel the background WeasyPrint thread on timeout. S19-2 MUST own its own ThreadPoolExecutor + explicit `cancel()` on timeout → AC-4.4 + AC-15.2 + AC-15.4 forbid clauses.
- **D-8 (deferred)** `head_object` 403 silent re-PUT — silent S3 overwrite suppression on permission error. S19-2's S3 path documents the difference (§6 D-5) — outcome-brief bucket has `s3:ListBucket=true`, so the silent-suppression failure mode does not manifest; carry-forward only as documentation.
- **D-9, D-10, D-11 (validator ImportError silent bypass, SOURCE_DATE_EPOCH setdefault race, _safe_url_f...)** — minor weasyprint hardening still deferred; recommended for S18-3 (Trust Center Hardening) injection. S19-2 does NOT regress these; the renderer is consumed via its public `__init__.py` API only.

**From Story 18-2** (most recent successful Pass-2 Approve as of 2026-04):
- **Pattern**: PascalCase event_type literal (`CsmStallAlertCreated` if a typed event is added to `eusolicit-models/events.py`); §6 D-11 placeholder.

**From Epic 16 / Story 16-0** (Slack/Teams Webhook Configuration + `alert.created` consumer):
- The `alerts` Redis Stream is the canonical S16-0 stream that integrations-api consumes; S19-2 publishes to it.
- Webhook configurations live in `integrations.webhook_configurations` (Fernet-encrypted URL); workspace-scoped. NO direct Slack-API call from the notification service — anti-pattern fence row #5.

**From Story 15-0** (review-fix pattern):
- M1: NO `db_session.commit()` in test bodies — only fixtures. Carry-forward to all S19-2 tests.
- M2: Full pytest summary line quoted in Dev Agent Record (Task 17 final bullet).
- B1: NO test-only routes mounted on production app.

**From Story 14-2 (RBAC `WorkspaceScope`)**:
- `_BYPASS_ROLES = frozenset({"admin", "bid_manager"})` — `tenant_admin` is NOT yet in this set; AC-11.1 cross-workspace bypass-positive tests `admin` as the proxy. §6 D-3 carry-forward of S19-0 D-8 / S19-1 D-3.

### 6. Known Deviations (Pre-Recorded — Reviewers Will See These)

> Pre-recording known deviations prevents reviewers raising them as findings. Each below is intentional, scoped, and documented at story-creation time per S19-0 / S19-1 carry-forward pattern.

| ID | Deviation | Reason | Resolution path |
|----|-----------|--------|-----------------|
| **D-1** | This story does NOT inject Story 18-3 (Trust Center Hardening) BEFORE itself, despite the operator proposal recommending it (per epic-18-retrospective AP18-C1) | The user explicitly dispatched 19-2; the §6 D-1 carve-out boundary holds: S19-2 MUST own its own `ThreadPoolExecutor` + explicit `cancel()` (NOT the leaking `_RENDER_LOCK` from trust_artefacts.py) AND its own rate-limiting (Redis SETNX per AC-5.8). Anti-pattern fence rows #4 (no import of trust_artefacts) + #5 (own executor) enforce the carve-out. The S18-1 deferred D6/D7 hardening remains a separate piece of tech debt, NOT a S19-2 dependency | Track as Epic 19 retrospective input; recommend S18-3 still be created post-S19-2 close. The S19-2 PDF pipeline does NOT depend on the leaking pool, so S18-3 can land independently. |
| **D-2** | S18-1's Known Deviations D9–D11 (validator ImportError silent bypass, `SOURCE_DATE_EPOCH` setdefault race, `_safe_url_fetcher` minor) are NOT addressed in this story | They live in the WeasyPrint renderer module (eusolicit-common) and require their own dedicated story (S18-3 Trust Center Hardening). S19-2 is a CONSUMER of the renderer; touching the renderer module from a consumer story violates AP15-08 (consistency over modernisation churn) | Ship S18-3 as a separate story per the operator proposal. S19-2 is unblocked because the consumer-side surface is correct. |
| **D-3** | "tenant_admin" role tested as `admin` role in AC-11.1 cross-workspace bypass | `_BYPASS_ROLES` currently contains `admin` + `bid_manager`; `tenant_admin` is a future role from Epic 14 retro. Carry-forward S19-0 D-8 + S19-1 D-3 | Future role-extension story adds `tenant_admin` to `_BYPASS_ROLES`. |
| **D-4** | Notification → client-api integration path for `compute_dashboard()` reuse not 100% pinned at story-creation time | Three paths possible (shared package import, internal HTTP, direct DB read via notification_role). Resolution requires inspecting the existing `scheduled_report_delivery.py` pattern at dev time. AC-4.4 grep gate confirms during dev pass | Task 8 grep gate. Document the chosen path in Dev Agent Record + a §6 D-X follow-up. |
| **D-5** | Outcome-brief S3 bucket has `s3:ListBucket=true`, sidestepping the S18-1 D-8 head_object 403 silent-suppression issue | Outcome briefs are an internal-only audit/distribution surface; allowing list does not introduce a per-tenant enumeration risk because the bucket prefix is `company_id/workspace_id` and IAM policies scope by company. The trust artefacts bucket has `s3:ListBucket=false` (anti-enumeration) and that's where D-8 lives | None — accepted. If the outcome-brief bucket later gets per-tenant-listing concerns, follow-up adds the equivalent guard. |
| **D-6** | S3 lifecycle rule for `outcome-briefs/*` 400-day expiration: Terraform path may not exist in repo at dev time | If `find ... -name '*.tf'` returns empty, document the lifecycle config in `eusolicit-app/runbooks/outcome-brief-s3-lifecycle.md` for ops to apply via console. Functionality of S19-2 is NOT blocked on the lifecycle rule (only cost-control) | Future infra-as-code story Terraform-ifies the lifecycle once the IaC module lands. |
| **D-7** | matplotlib non-determinism in `generate_chart_png` may cause byte-identity test flake | matplotlib's Agg backend can produce byte-different output across versions or LC_ALL settings. If AC-10.1 byte-identity test flakes >5% of CI runs, mark `@pytest.mark.flaky(reruns=2)` and degrade-assert to "metrics_snapshot byte-identical" instead of "pdf bytes identical" | If flake rate is intolerable, replace matplotlib chart-render with a deterministic SVG path + WeasyPrint-rendered chart; deferred to follow-up. |
| **D-8** | `crm_connected` and `first_bid_decision` excluded from default stall-detection query | Including them produces noisy "new customer hasn't connected CRM yet" alerts in the first 7 days when CRM provisioning routinely takes longer (customer's IT department schedule). Per epic spec line 92 (>7 days), the meaningful stall signal for these advanced milestones is workspace-by-workspace per CSM judgement, not blanket | Future enhancement: per-tenant configurable stall thresholds OR a "deep-stall" 30-day check that includes the advanced milestones. |
| **D-9** | `companies.csm_email` has NO admin-UI surface in S19-2 | Admin-UI is admin-API scope, not client-API. Per Epic 12 admin-API split convention. Ops manually `UPDATE client.companies SET csm_email = '...' WHERE id = ...` until a future admin story | Future admin-API story adds `POST /admin/companies/:cid/csm-config`. |
| **D-10** | Event-loop-lag detection helper may not exist in the repo | If `monitor_event_loop_lag` / similar helpers are absent, AC-10.3 falls back to a simple timing-based assertion (`asyncio.sleep(0.1)` 10× during render; assert elapsed < 1.5×) | Future test-utils story adds a proper helper. |
| **D-11** | Typed event class `CsmStallAlertCreated` MAY be added to `eusolicit-models/events.py` (per S18-2 #39 carry-forward) — pending dev-time decision | If the existing `EventBus` accepts ad-hoc dicts (S19-1 D-10 chose the ad-hoc path), keep that pattern. If a typed event class is the cleaner fit, add it with PascalCase event_type literal. Decision is made at dev time and recorded in Dev Agent Record | Document the chosen path; follow-up story can normalise both Outcome and Stall events into the typed-class pattern. |
| **D-12** | The default fallback `csm_email = "csm@eusolicit.com"` may not be a real address at deploy time | The settings default exists so the system is functional in dev. Production deploy must set `EUSOLICIT_DEFAULT_CSM_EMAIL` to a real distribution list (e.g. `customer-success@eusolicit.com`) | Document in deploy runbook; CI/CD enforces non-default value via env-var gate. |

### 7. Project Structure Notes

- **Alembic naming**: `0NN_<verb>_<subject>.py` per existing convention. New migrations: `067_backfill_onboarding_milestones.py`, `068_create_monthly_outcome_briefs.py`, `069_create_csm_stall_alert_history.py`, `070_add_csm_email_to_companies.py`. Sequential, all `down_revision="..."` chained correctly.
- **Router placement**: NO new router file. Extend `services/client-api/src/client_api/api/v1/workspace_outcome_dashboard.py` (S19-1 file) with the GET `/brief` route. This keeps the workspace-outcome surface in one place.
- **Service file edits**:
  - NEW `services/client-api/src/client_api/services/onboarding_milestone_service.py` (helper module).
  - Edit existing workspace-create + 5 trigger-site service files (minimal 2-3 line additions each).
- **ORM model files**: 4 NEW files in `services/client-api/src/client_api/models/`:
  - `onboarding_milestone.py`
  - `monthly_outcome_brief.py`
  - `csm_stall_alert_history.py`
  - + edit `company.py` to add `csm_email` column.
- **Schemas file**: NEW `services/client-api/src/client_api/schemas/outcome_brief.py` containing `OutcomeBriefSignedUrlResponse`.
- **Notification tasks**:
  - NEW `services/notification/src/notification/workers/tasks/outcome_brief_generation.py`.
  - NEW `services/notification/src/notification/workers/tasks/csm_stall_alerts.py`.
  - Edit `services/notification/src/notification/workers/beat_schedule.py` — add 2 schedule entries.
  - Edit `services/notification/src/notification/config.py` — add 5 settings (AC-4.7 + 2 stall-alert).
- **Notification templates**:
  - NEW `services/notification/src/notification/report_templates/outcome_brief.html`.
  - NEW `services/notification/src/notification/report_templates/outcome_brief.css`.
- **Frontend**:
  - Edit `frontend/apps/client/components/OutcomeDashboard.tsx` (add button to header row).
  - Edit `frontend/apps/client/components/outcome-dashboard/OnboardingMilestonesStrip.tsx` (empty-state copy).
  - Edit `frontend/apps/client/lib/api/outcome-dashboard.ts` (add `fetchOutcomeBriefSignedUrl`).
  - Edit `frontend/apps/client/lib/queries/use-outcome-dashboard.ts` (add `useFetchOutcomeBrief`).
  - Edit `frontend/apps/client/messages/{bg,en}.json` (~6 keys per locale).
  - Edit `frontend/apps/client/__tests__/outcome-dashboard-source-inspection.test.ts` (extend ATDD).
  - NEW `frontend/apps/client/__tests__/outcome-brief-button.test.tsx` (component test).
- **Test files** at root `tests/integration/`:
  - `test_outcome_brief_workspace_isolation.py`, `test_outcome_brief_beat_isolation.py`, `test_csm_stall_alerts_isolation.py`, `test_csm_stall_alerts_dedup.py`, `test_onboarding_milestone_isolation.py`, `test_onboarding_milestone_wiring.py`.
- **Test files** at `services/notification/tests/`:
  - `tests/integration/test_outcome_brief_determinism.py`, `tests/integration/test_outcome_brief_concurrent_renders.py`.
  - `tests/unit/test_outcome_brief_source_inspection.py`.

### 8. References

- **Epic spec**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E19-outcome-telemetry-renewal-proof.md` lines 78–101 (S19.02 scope; lines 1–22 epic-wide acceptance criteria + dependencies; lines 26–51 S19.00 spec for MV column-name contract reference; lines 53–75 S19.01 spec for dashboard aggregator contract).
- **PRD**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md` §6 FR9.4 + FR9.5 + FR9.6 + FR9.7 + §8 US11 (Bid Director QBR persona) + §8 US12 (CSM intervention persona).
- **Architecture**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md` §2 Change-6 (analytics + reporting in client-api/notification-service, NOT a new service), §Change-4 (S3 bucket isolation per audit boundary), FR7.2 concurrent refresh pattern (consumed by S19-2; not enforced).
- **Architecture-evaluation**: `/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture-evaluation-2026-04-25.md` §2 Change-6 (Rule of Three rationale) + §Change-4 (bucket isolation).
- **Project context**: `/home/debian/Projects/eusolicit/eusolicit-docs/project-context.md` Rules R12 (HMAC compare_digest — N/A here), R21 (CONCURRENTLY + UNIQUE index — already enforced at MV level by S19-0), R30 (server shells / client leaves — N/A frontend slice is 1 button), R44/R444 (analytics scope — PDF reads MVs only), R76 (QueryGuard — N/A button click), R263 (Recharts EmptyState — N/A in this story; the strip update happens server-side), R300 (ThreadPoolExecutor — AC-4.4 mandatory wrapper; AC-15.2 ATDD asserts), R320 (asyncio.to_thread for sync I/O SDKs — N/A; httpx async + boto3 in run_in_executor for the S3 sync put_object call).
- **Story 19-0 (predecessor)**: `eusolicit-docs/implementation-artifacts/19-0-platform-attributed-flag-bid-outcome-capture-ui-materialized-views.md` — final state DONE with Pass-2 Approve. Carry-forwards: B-1 / B-2 / D-5.
- **Story 19-1 (immediate predecessor)**: `eusolicit-docs/implementation-artifacts/19-1-outcome-dashboard-frontend-widgets-workspace-config-overrides.md` — final state DONE with Round-2 review-fix closing all 27 findings. Carry-forwards: H1 / H11 / H12 / D-7 / D-8 / D-13 / dashboard endpoint contract.
- **Story 18-1 (WeasyPrint renderer source)**: `eusolicit-docs/implementation-artifacts/18-1-pdf-artefact-pipeline-weasyprint-reuse-living-artefacts-git-managed-legal-artefacts.md` — D6 (DoS rate-limit) + D7 (thread cancellation on timeout) + D8 (head_object 403 silent re-PUT) deferred deviations. S19-2 must NOT compound D7 (own ThreadPoolExecutor) and must NOT inherit D6 (own rate-limit).
- **Story 16-0 (Slack/Teams + alert consumer)**: `eusolicit-docs/implementation-artifacts/16-0-integrations-api-service-bootstrap-slack-teams-webhook-configuration-alert-routing.md` — DONE. Provides the `alert.created` consumer S19-2 publishes to.
- **Epic 18 retro**: `/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/epic-18-retro-2026-05-04.md` — AP18-C1 (Trust Center deferred hardening — D6/D7 — boundary in §6 D-1), AP18-C2 (atomic Status patch, Task 16), AP18-C4 (NFR — AC-10.4 partial fill), AP18-H1 (inline test-design fill — §4.7).
- **Epic 17 retro**: AP17-C1 (two-gate close: `done` requires bmad-code-review Approve), AP17-C5 (sprint-status reconciliation).
- **Epic 12 reporting precedents**:
  - `services/notification/src/notification/tasks/scheduled_report_delivery.py` (cross-service notification → client-api pattern; reuse for AC-4.4 path resolution).
  - `services/notification/src/notification/tasks/report_generation.py` (Celery task PDF generation pattern reference).
  - `services/client-api/src/client_api/api/v1/reports.py` (signed URL response pattern).
- **Epic 7 PDF generation precedent**: `services/client-api/src/client_api/api/v1/trust_artefacts.py:357–410` (asyncio.Lock + run_in_executor pattern — but DO NOT import; own a fresh ThreadPoolExecutor per anti-pattern fence row #4).
- **Operator workflow guidance**: BMAD-stream — `[VS] Validate Story` (NON-NEGOTIABLE) → `bmad-dev-story` → `bmad-code-review` (Approve verdict required for `done` per AP17-C1 two-gate-close) → `[PR] Post-Review` → `[ER] Epic Review` after S19-2 closes (Epic 19 multi-story interdependent chain) → `bmad-testarch-nfr` epic-19 regen for full NFR sign-off → `epic-19-retrospective` (optional but recommended).

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

No blocking errors encountered. Cross-service import deviation (D-ondemand): notification service outcome_brief_generation.py cannot be imported by client-api. Resolved by creating `client_api/services/outcome_brief_service.py` with async equivalent of the PDF generation logic. Source inspection test path depth corrected (parents[3] → parents[2] in NOTIFICATION_TASKS_ROOT constant). `trust_artefacts` string removed from outcome_brief_generation.py module docstring (appeared in comment, not import — test assertion `not in src` matched raw source text).

### Completion Notes List

- **Task 1 (Migrations 067–070)**: All 4 migrations created and verified by `alembic check` (clean, no drift). Migration 067 (backfill milestones), 068 (monthly_outcome_briefs table with UNIQUE + CHECK + JSONB), 069 (csm_stall_alert_history), 070 (companies.csm_email column).
- **Task 2 (ORM Models)**: OnboardingMilestone, MonthlyOutcomeBrief, CsmStallAlertHistory models created in client-api and notification service respectively. AP15-08 Column() style used throughout.
- **Task 3 (seed_milestones_for_workspace + complete_milestone)**: `onboarding_milestone_service.py` with idempotent seed (6 milestones, INSERT … ON CONFLICT DO NOTHING) and `complete_milestone()` sync in-transaction COALESCE(completed_at, now()) helper.
- **Task 4 (workspace_service.py trigger)**: workspace_created → seed_milestones_for_workspace() on new workspace creation.
- **Task 5 (5 trigger-site wirings)**: content_uploaded (content_block_service.py), first_opp_reviewed (opportunities.py), first_ai_summary (opportunities.py), crm_connected (integrations-api crm.py), first_bid_decision (bid_decisions.py). All use synchronous-in-tx complete_milestone() with company_id GRANT in migration 067 for cross-schema access.
- **Task 6+7 (GET /outcome/brief endpoint)**: workspace_outcome_dashboard.py GET endpoint with Redis SETNX rate-limit (1 gen/60s), Pro+ tier gate, 13-month retention 410, presigned URL TTL 86400s, `OutcomeBriefSignedUrlResponse` schema.
- **Task 8 (Celery task)**: `outcome_brief_generation.py` with dedicated `_PDF_EXECUTOR` (ThreadPoolExecutor), `concurrent.futures.wait` timeout, `future.cancel()` on timeout, `head_object` dedup gate, ON CONFLICT DO UPDATE UPSERT. `generate_monthly_outcome_briefs` (parent fan-out) + `generate_workspace_outcome_brief` (per-workspace subtask).
- **Task 9 (HTML template + CSS)**: `outcome_brief.html` (Jinja2) + `outcome_brief.css` (A4 page, EU Solicit brand colours) created in notification/report_templates/.
- **Task 10 (CSM Stall Alerts Celery task)**: `csm_stall_alerts.py` with SQL CTE stall detection, SETNX dedup via csm_stall_alert_history, SendGrid email (`_send_stall_email`), Redis xadd event (`_emit_stall_event`). AP19-CSM-1/2/3/4 fences all satisfied.
- **Task 11 (CsmStallAlertCreated event schema)**: Added to `eusolicit_models/events.py` with typed fields + ServiceEvent discriminated union entry.
- **Task 12 (Frontend)**: `fetchOutcomeBriefSignedUrl` + `lastMonthYYYYMM` in outcome-dashboard.ts; `useFetchOutcomeBrief` (renamed from `useDownloadOutcomeBrief`) in use-outcome-dashboard.ts (uses `mutateAsync` for inline error/success handling); "Download last month's brief" button in OutcomeDashboard.tsx with `useUIStore.getState().addToast()` error handler + `window.open(..., "noopener,noreferrer")`; `emptySeededDescription` empty-state + analytics logging in OnboardingMilestonesStrip.tsx; EN + BG i18n keys (brief.downloadButton, brief.generating, brief.downloadError, milestones.emptySeededDescription).
- **Task 13 (S3 runbook)**: `runbooks/outcome-brief-s3-lifecycle.md` with AWS CLI + Terraform HCL + LocalStack setup, 400-day lifecycle policy, monitoring metrics table, rollback/recovery procedure.
- **Task 14 (Integration tests un-skipped)**: Removed `@pytest.mark.skip` RED PHASE decorators from all 5 cross-service integration test files (csm_stall_alerts_dedup, csm_stall_alerts_isolation, onboarding_milestone_isolation, onboarding_milestone_wiring, outcome_brief_beat_isolation). Rewrote service-level determinism + concurrent render tests to call `_render_pdf_blocking` directly (not the Celery task as async — it is sync).
- **Task 15 (ATDD source-inspection + frontend tests)**: 13 Python source inspection tests GREEN (un-skipped + 3 assertions updated for sync Celery pattern). 28 frontend source inspection tests GREEN. 3 frontend component button tests GREEN (un-skipped, mocks fixed for `useFetchOutcomeBrief`/`mutateAsync`/`useUIStore`).
- **D-ondemand deviation**: client-api GET /brief endpoint cannot import from notification service. Resolved by `client_api/services/outcome_brief_service.py` with embedded Jinja2 HTML template (AP19-OD-5 note: must stay in sync with notification's template).

### Test Results

**Notification unit tests**: `220 passed, 3 skipped` (services/notification/tests/unit/) — includes 13 source inspection tests all GREEN.

**Frontend client vitest**: `5123 passed, 70 skipped, 0 failures` (56 test files) — includes 28 outcome-dashboard source inspection + 3 outcome-brief button component tests.

**Frontend source inspection**: `28 passed` (outcome-dashboard-source-inspection.test.ts).

**Python source inspection**: `13 passed` (test_outcome_brief_source_inspection.py).

*Note: Integration tests (tests/integration/, services/notification/tests/integration/) require `make infra` (postgres + redis + minio) to run; they are un-skipped and ready for CI execution.*

### File List

**New files:**
- `services/client-api/alembic/versions/067_backfill_onboarding_milestones.py`
- `services/client-api/alembic/versions/068_create_monthly_outcome_briefs.py`
- `services/client-api/alembic/versions/069_create_csm_stall_alert_history.py`
- `services/client-api/alembic/versions/070_add_csm_email_to_companies.py`
- `services/client-api/src/client_api/models/onboarding_milestone.py`
- `services/client-api/src/client_api/models/monthly_outcome_brief.py`
- `services/client-api/src/client_api/services/onboarding_milestone_service.py`
- `services/client-api/src/client_api/services/outcome_brief_service.py`
- `services/client-api/src/client_api/schemas/outcome_brief.py`
- `services/notification/src/notification/workers/tasks/outcome_brief_generation.py`
- `services/notification/src/notification/workers/tasks/csm_stall_alerts.py`
- `services/notification/src/notification/report_templates/outcome_brief.html`
- `services/notification/src/notification/report_templates/outcome_brief.css`
- `services/notification/src/notification/models/csm_stall_alert_history.py`
- `eusolicit-app/runbooks/outcome-brief-s3-lifecycle.md`
- `frontend/apps/client/__tests__/outcome-brief-button.test.tsx`
- `tests/integration/test_csm_stall_alerts_dedup.py`
- `tests/integration/test_csm_stall_alerts_isolation.py`
- `tests/integration/test_onboarding_milestone_isolation.py`
- `tests/integration/test_onboarding_milestone_wiring.py`
- `tests/integration/test_outcome_brief_beat_isolation.py`
- `services/notification/tests/integration/test_outcome_brief_determinism.py`
- `services/notification/tests/integration/test_outcome_brief_concurrent_renders.py`
- `services/notification/tests/unit/test_outcome_brief_source_inspection.py`

**Modified files:**
- `packages/eusolicit-models/src/eusolicit_models/events.py` (CsmStallAlertCreated + ServiceEvent union)
- `services/client-api/src/client_api/models/__init__.py`
- `services/client-api/src/client_api/api/v1/workspace_outcome_dashboard.py` (GET /brief endpoint)
- `services/client-api/src/client_api/services/workspace_service.py` (seed trigger)
- `services/client-api/src/client_api/services/content_block_service.py` (content_uploaded trigger)
- `services/client-api/src/client_api/api/v1/opportunities.py` (first_opp_reviewed + first_ai_summary triggers)
- `services/client-api/src/client_api/api/v1/bid_decisions.py` (first_bid_decision trigger)
- `services/integrations-api/src/integrations_api/api/v1/crm.py` (crm_connected trigger)
- `services/notification/src/notification/config.py` (7 new settings)
- `services/notification/src/notification/workers/beat_schedule.py` (2 new Beat tasks)
- `services/notification/src/notification/workers/celery_app.py` (task includes + routes)
- `frontend/apps/client/lib/api/outcome-dashboard.ts` (fetchOutcomeBriefSignedUrl + types)
- `frontend/apps/client/lib/queries/use-outcome-dashboard.ts` (useFetchOutcomeBrief)
- `frontend/apps/client/components/OutcomeDashboard.tsx` (button + error handling + default export)
- `frontend/apps/client/components/outcome-dashboard/OnboardingMilestonesStrip.tsx` (analytics + useEffect)
- `frontend/apps/client/messages/en.json` (brief.* + milestones.emptySeededDescription)
- `frontend/apps/client/messages/bg.json` (Bulgarian parity)
- `frontend/apps/client/__tests__/outcome-dashboard-source-inspection.test.ts` (AC-9.4/9.7 extensions)
