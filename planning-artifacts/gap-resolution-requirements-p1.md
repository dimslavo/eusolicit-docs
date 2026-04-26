# EU Solicit — Gap Resolution Requirements: P1 Gaps

**Date**: 2026-04-22 | **Author**: Mary (Business Analyst) | **Status**: Draft for review
**Purpose**: Exhaustive requirements for P1 gaps (GAP-01, 02, 06, 07, 10) in the form needed for Requirements Brief v5, PRD v2, and scope-extension story drafting.
**Input**: `architecture-vs-requirements-gap-analysis.md`, `gap-coverage-matrix.md`, Solution Architecture v4, implementing story ACs.
**Convention**: FR-XX.YY (functional req), DM-XX (data model), API-XX (endpoint), NFR-XX (non-functional), EC-XX (edge case), NS-XX (new story proposal).

---

## GAP-01 — Task Orchestration & Workflow Engine

### Rationale (strategic)
Task orchestration is the backbone of Professional/Enterprise workflow value. Without it, the platform is a collection of tools rather than a governance system. Coverage is ✅ delivered via stories 10-5, 10-6, 10-7, 10-14, 10-15.

### Data model

**DM-01.1** `client.tasks` — core task entity. Columns:
- `id` UUID PK (gen_random_uuid)
- `company_id` UUID NOT NULL (tenant scope; no cross-tenant queries)
- `opportunity_id` UUID NULL (optional tie to a pipeline opportunity)
- `proposal_id` UUID NULL (optional tie to a proposal)
- `title` VARCHAR NOT NULL
- `description` TEXT NULL
- `assigned_to` UUID NULL — soft link to `client.users.id`
- `priority` ENUM {P1, P2, P3, P4}, default P2
- `status` ENUM {pending, in_progress, blocked, completed}, default pending
- `due_date` TIMESTAMPTZ NULL
- `completed_at` TIMESTAMPTZ NULL (auto-set on status→completed)
- `created_at`, `updated_at` TIMESTAMPTZ
- `deleted_at` TIMESTAMPTZ NULL (soft delete)

**DM-01.2** `client.task_dependencies` — DAG edges. Columns:
- `id` UUID PK
- `task_id` UUID FK → tasks.id ON DELETE CASCADE
- `depends_on_task_id` UUID FK → tasks.id ON DELETE CASCADE
- `dependency_type` ENUM {finish_to_start, finish_to_finish}, default finish_to_start
- UNIQUE `(task_id, depends_on_task_id)`

**DM-01.3** `client.task_templates` — reusable factories. Columns per story 10-7:
- `id` UUID PK; `company_id` FK; `name` VARCHAR(255); `description` TEXT
- `opportunity_type` ENUM {tender, grant, any}, default any
- `stages` JSONB (array 1..50) with `StageDefinition` schema: `{title, description, role, relative_days_before_deadline, dependency_indices[]}`
- `created_by`, `created_at`, `updated_at`, `deleted_at`
- Partial UNIQUE on `(company_id, name) WHERE deleted_at IS NULL`

**Template stage validation rules**:
- `dependency_indices[i]` must be strictly less than the stage's own index (enforces DAG by construction).
- `dependency_indices` contain no duplicates within a single stage.
- Stage count in [1, 50].
- `relative_days_before_deadline` in [0, 365].

### API contracts

**API-01.1 Tasks CRUD**
| Method | Path | Body / query | Success | Errors |
|--------|------|-------------|---------|--------|
| POST | `/api/v1/tasks` | `TaskCreateRequest` | 201 `TaskResponse` | 422 invalid assignee, 403 RBAC |
| GET | `/api/v1/tasks` | filters: company_id, opportunity_id, status, assigned_to, priority; limit, offset | 200 `TaskListResponse` | — |
| GET | `/api/v1/tasks/{id}` | — | 200 `TaskResponse` | 404 missing/cross-tenant |
| PATCH | `/api/v1/tasks/{id}` | partial fields | 200 `TaskResponse` | 404; 422 on invalid status transition (see FR-01.7) |
| DELETE | `/api/v1/tasks/{id}` | — | 204 | 404 |

**API-01.2 Task Dependencies**
- POST `/api/v1/tasks/{id}/dependencies` — body `{depends_on_task_id, dependency_type}`. 201 on insert; 422 on cycle detected (DFS pre-check).
- DELETE `/api/v1/tasks/{id}/dependencies/{dep_id}` — 204.

**API-01.3 Task Templates**
- POST, GET, GET /{id}, PATCH /{id}, DELETE /{id} on `/api/v1/task-templates` (RBAC: bid_manager for write, any member for read).
- **POST `/api/v1/task-templates/{id}/apply`** — body `{opportunity_id}`. Creates N tasks + M dependencies atomically; returns 201 with `{created_task_ids, created_dependency_ids, warnings[]}`.

### Functional requirements

**FR-01.1** Tasks have four status values (pending, in_progress, blocked, completed) and four priorities (P1–P4).

**FR-01.2** `assigned_to` at creation or update must resolve to an active member of the same company; otherwise the request returns 422.

**FR-01.3** On PATCH with `status=completed`, the server sets `completed_at = now()` atomically.

**FR-01.4** DELETE is soft: `deleted_at = now()`; queries filter `deleted_at IS NULL`.

**FR-01.5** **DAG validation (cycle rejection)**: before inserting a dependency `(A, B)`, run a directed graph reachability check starting at B; if A is reachable from B, reject with 422 and `detail="Cycle detected"`. Multi-hop cycles (A→B→C→A) are rejected.

**FR-01.6** **Finish-to-start gate**: transitioning `status→completed` is rejected with 422 when any upstream finish_to_start dependency is not `completed`.

**FR-01.7** **Finish-to-finish gate**: transitioning `status→completed` is rejected with 422 when any upstream finish_to_finish dependency is neither `completed` nor `in_progress`.

**FR-01.8** **Downstream auto-unblock**: on task completion, every downstream task whose finish_to_start upstreams are all completed transitions from `blocked` to `pending` within the same transaction.

**FR-01.9** **Template application semantics** (single DB transaction):
1. Resolve template by `(id, company_id, deleted_at IS NULL)` else 404.
2. Resolve opportunity via `opportunity_scope_service.ensure_opportunity_visible()` else 404.
3. Reject with 422 when opportunity.deadline IS NULL.
4. Emit non-blocking warning when template.opportunity_type ≠ opportunity.opportunity_type (and template type ≠ 'any').
5. For each stage i: insert a Task with `due_date = opportunity.deadline − stage.relative_days_before_deadline days`; warn if due_date is past but still create.
6. For each stage with dependency_indices, insert TaskDependency rows (always `finish_to_start`).
7. Any task with ≥1 finish_to_start upstream starts in `blocked`; tasks with no upstreams start in `pending`.
8. Commit. Write one audit row (`entity_type=task_template_application`, `action=apply`).

**FR-01.10** **Visibility**: task_templates filter uses `IN (requested, 'any')` so 'any' templates are visible to both tender and grant queries.

**FR-01.11** **RBAC (write operations)**: bid_manager role on company is required for POST/PATCH/DELETE on task_templates; POST/PATCH/DELETE on tasks require the caller to be either the task's company_admin, bid_manager, the creator, or the `assigned_to`.

**FR-01.12** Audit: every create/update/delete on tasks, task_dependencies, task_templates, and every template apply writes an audit row.

### NFRs

- **NFR-01.1** Cycle check traversal bounded to tasks of the same company; P95 latency < 50ms for graphs up to 500 tasks.
- **NFR-01.2** Dependency-list endpoint returns in O(1) queries (no N+1) via eager loading.
- **NFR-01.3** Template apply is atomic — partial failure (e.g., DB error on task 3 of 5) rolls back all inserts and writes no audit row.

### Negative paths

- Cross-tenant read/write returns 404 (existence-leakage safe).
- Duplicate dependency `(task_id, depends_on_task_id)` returns 409.
- Self-dependency `task_id = depends_on_task_id` returns 422.
- Template apply against non-existent opportunity returns 404; apply against a cross-tenant opportunity returns 404 (same code to avoid leakage).
- Assigning a task to a deactivated user returns 422.

### Watchlist / scope-extension proposals

**NS-01.1** — **Task recurrence**. Tasks today are single-occurrence. Many workflow items are periodic (e.g., "weekly progress meeting," "monthly budget review"). Propose `recurrence_rule` column (RFC 5545 RRULE string) + a Celery beat worker that materializes the next instance on completion of the current. New story: **S13.10 — Task recurrence (RRULE) support**.

**NS-01.2** — **Subtasks / task hierarchy**. Tasks are flat today. For complex bids, sub-task breakdowns are useful. Propose `parent_task_id` NULL column (self-FK) + completion-rollup logic (parent auto-completes when all subtasks are complete). New story: **S13.11 — Subtask hierarchy + rollup**.

**NS-01.3** — **Task attachments**. No file attachment model. Users would paste links in `description` today, which is poor governance. Propose `task_attachments` table with S3 references (reuse 6-6 upload plumbing). New story: **S13.12 — Task attachments with ClamAV-scanned S3 storage**.

**NS-01.4** — **Time-zone handling**. `due_date` is stored in UTC but surfaced as a timestamp. For multi-timezone teams, deadline display should respect each user's preferred timezone (stored on `client.users`). Propose: add `timezone` column to users + frontend translation. New story: **S13.13 — User timezone preference with per-user deadline rendering**.

**NS-01.5** — **Bulk operations**. No bulk-delete, bulk-reassign, bulk-status-change endpoints. Power users managing 50+ tasks need these. New story: **S13.14 — Bulk task operations API + UI**.

**NS-01.6** — **Parametric task templates**. Templates today are fixed stage sets. A consulting firm managing 5 clients wants "Client X adaptation" parameters (e.g., variable stage count based on bid value threshold). Propose template variables with simple substitution. Consider deferring — complexity/benefit ratio is unclear. Leave as "open design question" in PRD v2.

**NS-01.7** — **Overdue notification cadence**. Story 9-10 includes task overdue notifications but no explicit cadence rules (one-shot? daily? escalating?). Verify current implementation, expose configurable cadence in user preferences. New story (if absent): **S13.15 — Overdue task notification cadence preferences**.

---

## GAP-02 — Approval Gates

### Rationale (strategic)
Enterprise buyers require governance sign-off before external submission. Without approval gates, the platform is not deployable into regulated organizations. Coverage is ✅ delivered via stories 10-8, 10-9, 10-16.

### Data model

**DM-02.1** `client.approval_workflows`:
- `id` UUID PK; `company_id` FK; `name` VARCHAR(255); `description` TEXT
- `is_default` BOOLEAN default false
- `created_by` UUID; `created_at`, `updated_at`, `deleted_at` TIMESTAMPTZ
- Partial UNIQUE on `(company_id, name) WHERE deleted_at IS NULL`
- Partial UNIQUE on `(company_id) WHERE is_default = TRUE AND deleted_at IS NULL` — at most one default per company.

**DM-02.2** `client.approval_stages`:
- `id` UUID PK; `workflow_id` FK (CASCADE)
- `name` VARCHAR(255); `order` INT ≥ 1
- `required_role` ENUM reuses `client.proposal_collaborator_role` (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only)
- `auto_advance` BOOLEAN default false
- UNIQUE `(workflow_id, order)` — ordinals unique per workflow

**DM-02.3** `client.approval_decisions`:
- `id` UUID PK; `proposal_id` FK (CASCADE); `stage_id` FK (RESTRICT, so stage deletion is blocked if decisions exist)
- `decision` TEXT CHECK IN (approved, rejected, returned_for_revision)
- `comment` TEXT NULL
- `decided_by` UUID
- `created_at` TIMESTAMPTZ

**DM-02.4** `client.proposals.approved_at` — TIMESTAMPTZ NULL. Set on final-stage approval; never reset.

### Event contract

**`ApprovalDecided`** (Redis Stream `eu-solicit:approvals`) — consumed by Notification Service:
```
event_type: "ApprovalDecided"
proposal_id: UUID str
approval_id: UUID str         # approval_decisions.id
proposal_owner_id: UUID str   # proposals.created_by; fallback: first bid_manager on proposal
decision: "approved" | "rejected"   # returned_for_revision does NOT emit
decider_id: UUID str
decision_note: str | None
source_service: "client-api"
tenant_id: UUID str (company_id)
correlation_id: str
```

### API contracts

**API-02.1 Workflow CRUD** (`/api/v1/approval-workflows`)
- POST (bid_manager) — body `{name, description, stages[], is_default}`; `stages[].order` must be contiguous `{1..N}`; returns 201 with eager-loaded stages.
- GET (any member) — paginated list, ordered `is_default DESC, name ASC`.
- GET /{id} — 404 safe for cross-tenant/soft-deleted.
- PATCH /{id} (bid_manager) — stage replacement blocked with 409 if decisions exist on any stage.
- DELETE /{id} (bid_manager) — soft-delete; 409 if decisions exist.
- POST /{id}/set-default (bid_manager) — atomic swap, returns `{id, is_default: true, previous_default_id}`.

**API-02.2 Decision engine** (`/api/v1/proposals/{proposal_id}/approvals`)
- **POST `/decide`** — body `{stage_id, decision, comment}`; requires proposal-collaborator role ≠ read_only.
- **GET `/history`** — chronological decision list; all 5 roles may view (including read_only).

### Functional requirements

**FR-02.1** Workflow names must be unique per company among live rows. Rename to an existing name returns 409.

**FR-02.2** At most one workflow per company can have `is_default=true`. Setting `is_default=true` atomically demotes the previous default within the same transaction.

**FR-02.3** `stages[].order` must form the contiguous set `{1, 2, …, len(stages)}` with no gaps, no duplicates, no zero-index. Rejected with 422 at Pydantic layer.

**FR-02.4** **Decision role gate**: for non-admin callers, the proposal-collaborator role must equal the stage's `required_role` (or be covered by admin bypass). Mismatch returns 403 and writes an `access_denied` audit row.

**FR-02.5** **Prior-stage gate**: a decision on stage S is rejected with 409 if any prior stage (order < S.order in the same workflow) does not have `approved` as its latest decision.

**FR-02.6** **Mixed-workflow guard**: a proposal is "established" on a workflow by its earliest decision row. Subsequent decisions on stages of a different workflow return 409 `"Proposal already in a different approval workflow"`.

**FR-02.7** **Multi-decision-per-stage**: the DB does not unique-constrain `(proposal_id, stage_id)`. A stage can be decided multiple times; gates read the **latest** decision per stage via `DISTINCT ON (stage_id) … ORDER BY created_at DESC`.

**FR-02.8** **Comment requirement**: when decision ∈ {rejected, returned_for_revision}, `comment` must be non-empty after strip; otherwise 422.

**FR-02.9** **Final-stage approval**: when `stage.order == MAX(order)` AND `decision=approved`, set `proposals.approved_at = now()` atomically, guarded by `WHERE approved_at IS NULL` (idempotent; never resets).

**FR-02.10** **Event emission** happens post-commit, inside try/except — emission failure does not bubble up or roll back. Emission is skipped for `returned_for_revision` and when `proposal_owner_id` cannot be resolved.

**FR-02.11** **Soft-delete of workflows** retains stage rows (stages are not cascade-deleted) so historical decisions keep resolving their `stage_id` FK.

**FR-02.12** Stage modification on a workflow with recorded decisions returns 409 — stages are append-only once any proposal has used the workflow.

### NFRs

- **NFR-02.1** P95 decision POST latency < 200ms including role check, prior-stage gate, mixed-workflow guard, event publish.
- **NFR-02.2** Workflow list GET uses `selectinload` to eager-load stages; no N+1.
- **NFR-02.3** Event schema validates against `eusolicit_models.events.ApprovalDecided` via `TypeAdapter` — contract test required.

### Negative paths

- 404 on missing / cross-tenant / soft-deleted workflow, stage, or decision.
- 403 on role mismatch (audit access_denied).
- 409 on duplicate name, prior-stage-not-approved, mixed-workflow, stages-with-decisions modification, decisions-reference-workflow-on-delete.
- 422 on non-contiguous stage orders, empty comment for reject/return-for-revision, decision type not in enum.

### Watchlist / scope-extension proposals

**NS-02.1** — **Delegation / substitute reviewer**. If the designated approver is OOO, there is no way to delegate. Current flow blocks indefinitely. Propose: temporary delegation entries in `client.approval_delegations(user_id, stage_role, delegate_user_id, valid_from, valid_to)` consulted by the role gate. New story: **S13.16 — Approval delegation**.

**NS-02.2** — **Parallel-stage execution**. All stages today execute serially (prior-stage gate). Real organizations often run legal and finance review in parallel. Propose: add `parallel_group: int | None` on approval_stages — stages sharing a parallel_group can be decided independently; the next sequential stage waits for all parallel-group stages to be approved. New story: **S13.17 — Parallel approval-stage groups**.

**NS-02.3** — **SLA tracking on pending approvals**. No timer on how long a decision has been pending. Enterprise SLAs on approval turnaround exist. Propose: compute "pending since" on latest-decision absence; surface in notification digests and dashboards. Small story: **S13.18 — Approval SLA tracking + escalation notifications**.

**NS-02.4** — **Approver identity at decision time**. `decided_by` is captured, but if the user's role on the company changes after the fact, the audit record doesn't preserve *what role they held when deciding*. Propose: snapshot `decider_role_at_decision` on approval_decisions. Minor migration + service-layer change. New story: **S13.19 — Persist decider role snapshot on approval_decisions**.

**NS-02.5** — **Policy-change impact on in-flight proposals**. When a workflow's stages are changed (currently blocked if decisions exist), there's no "branch" mechanism — the company is stuck. Propose: `supersedes_workflow_id` on new workflow + migration helper that assigns in-flight proposals to the new workflow at the company's discretion. Complex; design-spike rather than immediate story.

---

## GAP-06 — Guided Submission Workflows ⚠️ PARTIAL

### Rationale (strategic)
Self-service bidding depends on the platform guiding users through external submission portals. Without curated, reviewed guides and interactive checklists, the platform ships raw AI output to users and loses the advisory role. This is the weakest link in the post-generation UX.

### Current delivery (retained, not re-specified)

- `pipeline.submission_guides(id, opportunity_id, steps JSONB, reviewed BOOL, source_portal, generated_by, created_at)` — auto-generated per opportunity (story 5-8).
- Surfaced read-only on opportunity detail page (6-5, 6-11).

### Gaps in delivery (new requirements)

**NEW DM-06.1** `admin.submission_guide_templates` — admin-curated portal templates:
- `id` UUID PK
- `portal_key` VARCHAR(64) NOT NULL — e.g., "aop_bg", "ted_eu", "eu_funding_ted"
- `opportunity_type` ENUM {tender, grant, any}
- `country_code` CHAR(2) NULL (ISO-3166-1 alpha-2; NULL = EU-wide)
- `version` INT NOT NULL default 1
- `name` VARCHAR(255)
- `steps` JSONB — array of step objects: `{title, description, portal_url, expected_artifacts[], required: bool, order}`
- `is_published` BOOLEAN default false — draft vs. live
- `published_at` TIMESTAMPTZ NULL
- `created_by`, `updated_by` UUID; `created_at`, `updated_at` TIMESTAMPTZ
- `deleted_at` TIMESTAMPTZ NULL
- UNIQUE `(portal_key, version) WHERE deleted_at IS NULL`
- Partial UNIQUE on `(portal_key) WHERE is_published = TRUE AND deleted_at IS NULL` — at most one published version per portal.

**NEW DM-06.2** `pipeline.submission_guides` — add columns:
- `template_id` UUID NULL — FK to `admin.submission_guide_templates.id` (nullable: guides generated before any template exists have NULL).
- `reviewed_by` UUID NULL
- `reviewed_at` TIMESTAMPTZ NULL
- `review_notes` TEXT NULL
- `override_steps` JSONB NULL — admin-edited step list; when non-null overrides the AI `steps` at render time.
- `review_status` ENUM {pending, approved, revised} default pending.

**NEW DM-06.3** `client.submission_progress` — per-user interactive checklist state:
- `id` UUID PK
- `proposal_id` UUID FK (CASCADE)
- `user_id` UUID FK
- `step_index` INT NOT NULL (0-based position in the rendered guide steps)
- `step_hash` VARCHAR(64) — SHA-256 of the step title + order, so reindexing a revised guide does not silently lose state
- `completed_at` TIMESTAMPTZ NOT NULL
- `notes` TEXT NULL
- UNIQUE `(proposal_id, user_id, step_index)`

### API contracts

**API-06.1 Admin template CRUD** (`admin-api`, `/api/v1/submission-guide-templates`, platform_admin role)
- POST, GET (list with filters: portal_key, country_code, opportunity_type, is_published), GET /{id}, PATCH /{id}, DELETE /{id} (soft), POST /{id}/publish (sets `is_published=true, published_at=now()`; demotes previous published version for same portal).

**API-06.2 Admin review queue** (`admin-api`, `/api/v1/submission-guides/review`, platform_admin)
- GET — list pending-review auto-generated guides with filters (portal, date range, reviewed_status).
- POST /{guide_id}/approve — body `{review_notes?}`; flips `reviewed=true, review_status=approved, reviewed_by, reviewed_at`.
- POST /{guide_id}/override — body `{override_steps, review_notes?}`; persists the manual edit and flips `review_status=revised, reviewed=true`.
- POST /{guide_id}/reject — body `{review_notes}`; sets `review_status=rejected` and triggers re-generation on next crawl.

**API-06.3 Client checklist** (`client-api`, paid tier)
- GET `/api/v1/proposals/{proposal_id}/submission-guide` — returns the effective guide: if `override_steps` present use it; else AI `steps`; includes per-user completion state from submission_progress.
- POST `/api/v1/proposals/{proposal_id}/submission-progress` — body `{step_index, step_hash, notes?}`; upserts completion for the caller.
- DELETE `/api/v1/proposals/{proposal_id}/submission-progress/{step_index}` — unchecks.

### Functional requirements

**FR-06.1 Template publishing** — at most one published version per portal_key at any time. `POST /publish` atomically demotes the prior published version (sets `is_published=false`) and promotes the target in a single transaction.

**FR-06.2 Guide resolution order** (client render): override_steps → matched template's steps (by portal_key + country_code + opportunity_type) → AI-generated steps → empty state with "No guide available" message.

**FR-06.3 Review gating** — Free and Starter tiers see *any* guide; Professional+ tiers see only guides with `review_status ∈ {approved, revised}` by default, with a user preference toggle `show_unreviewed_guides` for power users. (This makes review-queue backlog not user-visible for the majority.)

**FR-06.4 Step hash consistency** — submission_progress.step_hash is computed at write time as `sha256(step.title || "::" || step.order)`. If the guide is revised after the user completed a step, the UI shows the old step as "completed but content changed" and prompts re-confirmation.

**FR-06.5 Per-proposal completion** — a proposal's submission completion state is per-user (not per-company). Two collaborators on the same proposal may each have their own completion state. Aggregate completion surfaced at company level as `any_user_completed`.

**FR-06.6 Admin audit** — every template create/update/publish/delete and every guide review action writes an audit row in `shared.audit_log` with actor = platform_admin user, entity = template/guide.

**FR-06.7 Review SLA surfacing** — guides pending review for > 48h appear in an admin-dashboard "stale review" widget (not a hard SLA; visibility only).

**FR-06.8 AI regeneration on reject** — POST /reject re-enqueues the opportunity into the enrichment queue so story 5-8's Celery task regenerates.

### NFRs

- **NFR-06.1** Guide resolution endpoint P95 < 150ms for a proposal with ≤ 30 steps and ≤ 200 completion rows.
- **NFR-06.2** Per-portal published-version atomic swap must hold a row-level lock on the partial-unique index to prevent race conditions during concurrent POST /publish calls.
- **NFR-06.3** Template search by `(portal_key, country_code, opportunity_type)` must use a composite index.

### Negative paths

- 404 on missing template / guide / cross-tenant proposal.
- 409 on concurrent publish to same portal_key.
- 422 on step_hash mismatch (UI resubmits completion).

### Proposed new stories (confirmed in-scope)

- **S13.1 — Admin API: Submission Guide Template CRUD** (DM-06.1 + API-06.1)
- **S13.2 — Admin API + UI: AI-generated guide review queue** (API-06.2 + DM-06.2 columns + admin frontend)
- **S13.3 — Client UI: Interactive submission checklist with per-step acknowledgement + persistence** (DM-06.3 + API-06.3 + frontend)
- **S13.4 — Portal metadata & guide versioning schema migration** (DM-06.1 version column + publish semantics FR-06.1)

---

## GAP-07 — 14-Day Free Trial

### Rationale (strategic)
The trial is the primary conversion funnel from Free to paid. Coverage is ✅ delivered via stories 8-3, 8-5, 8-13, 9-11.

### Data model (current)

`client.subscriptions`:
- `stripe_subscription_id` VARCHAR NULL
- `tier` ENUM {free, starter, professional, enterprise}
- `status` ENUM — Stripe-aligned {active, trialing, past_due, canceled, incomplete, incomplete_expired}
- `trial_start`, `trial_end`, `trial_converted_at` TIMESTAMPTZ NULL
- `is_trial` BOOLEAN
- `started_at` TIMESTAMPTZ

### Functional requirements

**FR-07.1 Trial provisioning** — on new company registration, a background task creates a Stripe subscription on the Professional price with:
- `trial_end = now() + 14 days` (Unix timestamp)
- `payment_behavior = "default_incomplete"` (no payment method required)
- idempotency key `trial-provisioning-{company_id}`
Local subscription row is atomically updated: `tier="professional"`, `status="trialing"`, `trial_start=now()`, `trial_end=now()+14d`, `is_trial=true`.

**FR-07.2 Single-trial-per-company** — if `stripe_subscription_id IS NOT NULL`, the provisioning task skips with INFO log `trial_already_provisioned`.

**FR-07.3 Trial-feature access** — tier-gate middleware (`require_paid_tier`, `require_professional_plus_tier`, `require_enterprise_tier`) accepts `status IN ("active", "trialing")`. Trial Professional users get full Professional feature access.

**FR-07.4 Trial-will-end webhook** — on `customer.subscription.trial_will_end`, publish `trial.expiring` event to Redis Streams with `{company_id, trial_end, days_remaining: 3}`. No DB write.

**FR-07.5 Trial expiry downgrade** — on `customer.subscription.updated` or `customer.subscription.deleted` where local `is_trial=true` and incoming Stripe status ∈ {past_due, canceled, incomplete_expired}: force `tier=free, status=active, is_trial=false`. Emit `subscription.changed` event. This override applies ONLY when `is_trial=true`; paid `past_due` retains Stripe status for the existing 8-4 dunning flow.

**FR-07.6 Data preservation** — trial expiry downgrade touches only the subscription row. No rows in proposals, opportunities, users, companies, documents, etc. are deleted.

**FR-07.7 Event publishing order** — all subscription-changed events are published *after* `session.commit()`. Failures are WARN-logged, non-fatal.

**FR-07.8 Graceful degradation** — missing `stripe_customer_id`, missing `PROFESSIONAL_PRICE_ID`, or Stripe API error do NOT block registration; they log WARN/ERROR and skip provisioning. A retry job periodically reconciles subscriptions missing a `stripe_subscription_id` (see NS-07.1).

**FR-07.9 Audit** — trial provisioning, trial_will_end, and trial_expired each write audit rows with `action_type` prefixed `billing.`.

### NFRs

- **NFR-07.1** Registration response returns within 500ms — all Stripe calls are in a background task, not in the request path.
- **NFR-07.2** Stripe webhook idempotency: same `event.id` processed twice does not double-downgrade; the `webhook_events` table deduplicates.

### Negative paths

- Trial already provisioned → no-op (not an error).
- Stripe 5xx → ERROR log, swallowed; retry on next reconciliation pass.
- Non-trial subscription going `past_due` → raw Stripe status preserved; 8-4 dunning flow handles.

### Watchlist / scope-extension proposals

**NS-07.1** — **Trial provisioning reconciliation job**. If Stripe is down at registration, the company has `is_trial=false` and no trial at all. Propose a Celery beat job that finds companies with `subscriptions.tier='free' AND created_at > now()-interval '72 hours' AND stripe_subscription_id IS NULL` and retries trial provisioning. New story: **S13.20 — Trial provisioning reconciliation job**.

**NS-07.2** — **Abuse prevention**. Today, a single person can register multiple companies (different legal names, same payment method) and get unlimited 14-day trials. Propose:
- Fingerprint by `admin_user.email domain` + `company.tax_id` + `stripe_customer.payment_method.fingerprint` if present.
- Soft-block second trial on the same fingerprint (require payment method to start).
New story: **S13.21 — Trial abuse prevention (payment-method fingerprint)**.

**NS-07.3** — **Trial-to-paid mid-trial conversion**. Today, if a user upgrades during trial, the remaining trial days are not credited. This is documented in Architecture v4 §10.1 but the behavior may surprise users. Propose UX copy surfacing this clearly at the Stripe Checkout handoff. Minor frontend story: **S13.22 — Trial conversion UX copy (remaining-days disclosure)**.

**NS-07.4** — **Trial extension by admin**. No admin override to extend a trial by N days for high-value prospects on sales calls. Propose: admin-api endpoint `POST /admin/subscriptions/{id}/extend-trial` with body `{additional_days: int, reason: str}` that updates both Stripe and local subscription row; audited. New story: **S13.23 — Admin trial-extension override**.

**NS-07.5** — **Trial-end grace period**. The current downgrade is immediate on trial expiry. Propose a 24h grace period where the company retains Professional access with a persistent "Trial expired — upgrade now" banner, after which downgrade is applied. Reduces churn for users who were about to upgrade. Decision point; if accepted: **S13.24 — 24h trial-expiry grace period**.

---

## GAP-10 — Proposal Collaboration Model

### Rationale (strategic)
Multi-user editing with governance is the Professional-tier value prop. Coverage is ✅ delivered via stories 10-1 (collaborators), 10-3 (section locking), 10-4 (comments), 10-12 (UI), 10-13 (comments sidebar UI).

### Data model

**DM-10.1** `client.proposal_collaborators`:
- `id` UUID PK
- `proposal_id` FK (CASCADE); `user_id` FK (CASCADE)
- `role` ENUM `proposal_collaborator_role` {bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only}
- `granted_by` UUID NULL (soft link to users.id; NULL for system-seeded rows)
- `granted_at`, `updated_at` TIMESTAMPTZ
- UNIQUE `(proposal_id, user_id)`
- IX on `proposal_id`, `user_id`

**DM-10.2** `client.proposal_section_locks`:
- `id` UUID PK
- `proposal_id` FK (CASCADE); `section_key` TEXT; `locked_by` FK (CASCADE)
- `acquired_at`, `expires_at`, `updated_at` TIMESTAMPTZ
- UNIQUE `(proposal_id, section_key)` — the serialization point for `ON CONFLICT DO UPDATE`
- IX on `expires_at` (for the cleanup job), `proposal_id` (for enrichment JOIN)

**DM-10.3** `client.proposal_comments`:
- `id` UUID PK
- `proposal_id`, `version_id` FK (CASCADE); `section_key` TEXT
- `author_id` FK (RESTRICT — cannot lose comment authorship to user delete)
- `body` TEXT (1..10000 chars, DB CHECK)
- `parent_comment_id` self-FK NULL (CASCADE — thread roots drag replies with them)
- `resolved` BOOL default false; `resolved_by` FK (SET NULL); `resolved_at` TIMESTAMPTZ NULL
- `carried_forward_from_id` self-FK NULL (SET NULL)
- `created_at`, `updated_at` TIMESTAMPTZ
- CHECK `ck_proposal_comments_resolved_consistency`: `(resolved, resolved_by, resolved_at)` triple is either `(F, NULL, NULL)` or `(T, NOT NULL, NOT NULL)`.
- IXs: `(proposal_id)`, `(proposal_id, version_id, section_key, created_at)`, partial `(proposal_id, version_id, resolved) WHERE resolved=FALSE`, partial `(parent_comment_id) WHERE parent_comment_id IS NOT NULL`.

### API contracts

**API-10.1 Collaborator CRUD** (`/api/v1/proposals/{proposal_id}/collaborators`)
- POST (admin or proposal bid_manager) — body `{user_id, role}`; 409 duplicate, 422 non-company-member, 404 cross-tenant.
- GET — list.
- PATCH /{id} — role update.
- DELETE /{id} — 204.

**API-10.2 Section locking** (`/api/v1/proposals/{proposal_id}/sections/{section_key}`)
- POST /lock — acquires lock or refreshes existing lock for same holder; 423 with `SectionLockConflictDetail` body (NOT `{detail}`) on conflict; 15-min TTL.
- DELETE /lock — releases; 204; no-op if expired.

**API-10.3 Comments** (`/api/v1/proposals/{proposal_id}/comments`)
- POST — body `{version_id, section_key, body, parent_comment_id?}`.
- GET — filters: version_id, section_key, resolved; returns `{comments[], total, unresolved_count}`.
- PATCH /{id}/resolve, PATCH /{id}/unresolve — path-only.
- DELETE /{id} — author-only; 204.

### Functional requirements

**FR-10.1 Role levels** — 5 levels: bid_manager > technical_writer, financial_analyst, legal_reviewer > read_only. Write operations require ≥ technical_writer on the proposal; comment and history reads include read_only.

**FR-10.2 Collaborator authorization** — adding a collaborator requires caller = company admin OR caller holds proposal bid_manager role on that proposal.

**FR-10.3 Target-user tenant isolation** — `user_id` in POST must be an active member of the same company (row in `client.company_memberships` with `accepted_at IS NOT NULL`, user `is_active=true`). Cross-company assignment returns 422.

**FR-10.4 Section-lock semantics** — pessimistic lock, 15-minute TTL, one editor per section. Re-acquire by the same user refreshes the lock; different user gets 423 until expiry.

**FR-10.5 Lock cleanup** — Celery beat task every 60 seconds deletes expired rows (`WHERE expires_at < NOW()`). The index on `expires_at` makes this O(log n).

**FR-10.6 Comment carry-forward** — when a new proposal version is created, all **unresolved** comments from the prior version are copied to the new version, with `carried_forward_from_id` pointing at the original. Resolved comments are NOT carried forward (they stay on the old version for history).

**FR-10.7 Resolution consistency** — the DB CHECK enforces `resolved` boolean aligns with both `resolved_by` and `resolved_at` being set together or all three being null.

**FR-10.8 Unresolved count** — GET comments returns `unresolved_count` computed as `COUNT(*) FILTER (WHERE resolved=FALSE)` so the sidebar UI doesn't re-iterate.

**FR-10.9 Thread semantics** — `parent_comment_id` self-FK supports one-level threading (reply to a comment). Reply to a reply is flattened to the same root (UI responsibility).

**FR-10.10 Author-only delete** — DELETE by a non-author returns 403; admin and bid_manager do NOT bypass comment delete (immutability principle for audit).

**FR-10.11 Existence-leakage safety** — 404 on all cross-tenant / missing / soft-deleted resources, regardless of whether the caller has any proposal access.

### Event contracts

Collaboration mutations do NOT emit events directly (no external consumer today). `@mention` handling is NS-10.1.

### NFRs

- **NFR-10.1** Lock acquire P95 < 50ms (single `INSERT ... ON CONFLICT DO UPDATE` round-trip).
- **NFR-10.2** Comment list with `unresolved_count` P95 < 150ms on a proposal with ≤ 500 comments (covered by composite index).
- **NFR-10.3** Concurrent POST /lock from different users serialize on the UNIQUE constraint — exactly one 200, all others 423.

### Negative paths

- 403: non-authorized collaborator-add, non-author comment-delete, role-not-sufficient operation.
- 404: cross-tenant / missing resource, including soft-deleted.
- 409: duplicate collaborator on same proposal.
- 422: target user not in company, comment body out of range.
- 423: section-lock conflict with `SectionLockConflictDetail` body.

### Watchlist / scope-extension proposals

**NS-10.1** — **@mention → notification**. Comments support `parent_comment_id` threading but no mention-detection or notification dispatch. Propose:
- Lightweight `@[user_id]` syntax in comment body.
- On POST, parse mentions, publish `comment.mentioned` event to Redis Streams.
- Notification service delivers email / in-app notification via story 9-10 channel.
New story: **S13.25 — @mention detection + notification dispatch for proposal comments**.

**NS-10.2** — **Concurrent-edit UX for unlocked sections**. Pessimistic lock is coarse: if Alice locks section "Executive Summary" and Bob wants to edit a different sub-heading in the same section, he's blocked. Propose finer-grained lock on sub-section paths OR optimistic concurrency with conflict resolution UI. Complex — design spike first: **S13.26 — Section-lock granularity spike**.

**NS-10.3** — **Admin lock override**. Today no admin can force-release a stuck lock (e.g., user's browser crashed, 15min TTL still pending). Propose: admin/bid_manager force-release endpoint `DELETE /lock/force` with `reason: str` audited. New story: **S13.27 — Admin section-lock force-release**.

**NS-10.4** — **Lock-holder disclosure on 423** — currently included as `locked_by_full_name`. Verify UI surfaces this; if not, cosmetic story: **S13.28 — 423 UI with lock-holder and expiry timer**.

**NS-10.5** — **Session/logout lock release**. When a user logs out or their session expires, their held locks are NOT released — they wait for the 15-min TTL. Propose: on logout event (2-5 token revocation), enqueue a best-effort cleanup that deletes all locks held by that user. New story: **S13.29 — Lock cleanup on session revocation**.

**NS-10.6** — **Cross-company consortium scenarios**. Currently, a user in company A cannot be a collaborator on a proposal in company B (FR-10.3 enforces same-company). Consortium bidding would benefit from "shared proposal" cross-tenant grants. This is a significant scope expansion and likely belongs in a post-MVP epic. Document as open question in PRD v2 §7 Out-of-Scope with a roadmap mention.

**NS-10.7** — **Comment authorship snapshot**. If an author leaves the company and the user row is soft-deleted or email changed, the comment's `author_id` FK still resolves but the displayed `author_full_name` may show a re-provisioned name. Propose: snapshot `author_full_name_at_posted` on comments. Minor migration. **S13.30 — Comment authorship name snapshot**.

---

## Summary of proposed new stories from P1 exhaustive pass

| Story | Gap | Title | Priority |
|-------|-----|-------|----------|
| S13.1 | GAP-06 | Admin API: Submission Guide Template CRUD | P1 (gap-closing) |
| S13.2 | GAP-06 | Admin review queue: UI + API | P1 (gap-closing) |
| S13.3 | GAP-06 | Client UI: Interactive submission checklist | P1 (gap-closing) |
| S13.4 | GAP-06 | Portal metadata & guide versioning schema | P1 (gap-closing) |
| S13.10 | GAP-01 | Task recurrence (RRULE) | P2 (watchlist) |
| S13.11 | GAP-01 | Subtask hierarchy + rollup | P2 (watchlist) |
| S13.12 | GAP-01 | Task attachments | P3 (watchlist) |
| S13.13 | GAP-01 | User timezone preference + deadline rendering | P2 (watchlist) |
| S13.14 | GAP-01 | Bulk task operations API + UI | P3 (watchlist) |
| S13.15 | GAP-01 | Overdue task notification cadence preferences | P3 (verify-first) |
| S13.16 | GAP-02 | Approval delegation | P2 (watchlist) |
| S13.17 | GAP-02 | Parallel approval-stage groups | P2 (watchlist) |
| S13.18 | GAP-02 | Approval SLA tracking + escalation | P3 (watchlist) |
| S13.19 | GAP-02 | Persist decider role snapshot on decisions | P3 (audit robustness) |
| S13.20 | GAP-07 | Trial provisioning reconciliation job | P2 (robustness) |
| S13.21 | GAP-07 | Trial abuse prevention (payment-method fingerprint) | P2 (commercial) |
| S13.22 | GAP-07 | Trial conversion UX copy | P3 (polish) |
| S13.23 | GAP-07 | Admin trial-extension override | P2 (sales tooling) |
| S13.24 | GAP-07 | 24h trial-expiry grace period | P3 (decision required) |
| S13.25 | GAP-10 | @mention detection + notification | P2 (watchlist) |
| S13.26 | GAP-10 | Section-lock granularity spike | P3 (design spike) |
| S13.27 | GAP-10 | Admin section-lock force-release | P2 (ops tooling) |
| S13.28 | GAP-10 | 423 UI with lock-holder + expiry timer | P3 (polish; verify first) |
| S13.29 | GAP-10 | Lock cleanup on session revocation | P2 (correctness) |
| S13.30 | GAP-10 | Comment authorship name snapshot | P3 (audit polish) |

**Total new stories from P1**: 25 (4 gap-closing, 21 watchlist-driven). The 4 gap-closing stories (S13.1–S13.4) are mandatory to close GAP-06. The 21 watchlist-driven stories are recommended but optional — Deb to triage.

---

## Open decisions for Deb (before P2)

1. **Watchlist story triage** — do you want all 21 watchlist-driven stories pushed into the orchestrator backlog, or should I filter to only P2 and above (drops the 10 P3 items)?
2. **NS-01.6 parametric templates**, **NS-02.5 policy-change impact**, **NS-10.6 consortium cross-tenant** — these are design spikes rather than stories. Flag as PRD v2 §7 Out-of-Scope open questions, or draft as `spike-*` planning notes?
3. **NS-07.4 admin trial extension**, **NS-07.5 24h grace period** — commercial decisions. Proceed? Defer? I need a yes/no to include in Requirements Brief v5.
