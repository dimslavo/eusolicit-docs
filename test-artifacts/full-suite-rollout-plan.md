---
type: rollout-plan
relatedSummary: automation-summary-story-11-7.md
generatedBy: bmad-testarch-automate
lastSaved: '2026-05-09'
status: draft — awaiting Deb's per-phase approval
---

# Full-Scale Test Suite Rollout — P2 → P4

This plan extends the automation work after **P1** (story 11-7 RED→GREEN, complete in
`automation-summary-story-11-7.md`). Each phase is sized to be a self-contained
`/bmad-testarch-automate` run with its own deliverable. Run them in any order, but
**P2 first** is recommended because it cleanly maps to existing skipped tests with
known-good intent.

Sprint state at time of plan:
- 21/21 epics done, 191 stories done, 0 in-flight (per
  `eusolicit-docs/implementation-artifacts/sprint-status.yaml`).
- Test designs published only for epics 1–9, 11, 12. Epics 10, 13, 14–21 lack
  consolidated test design — gaps below are derived from code + sprint-status.

---

## P2 — Re-enable skipped E2E specs (cluster A)

**Goal:** Activate ~240 skipped Playwright tests across 7 spec files. Each test
either gets un-skipped + repaired, or downgraded to `test.fixme` with a written
reason.

### Spec inventory

| Spec | Total tests | Skipped | Anchor story | Implementation status |
|---|---:|---:|---|---|
| `e2e/specs/auth/auth-pages.spec.ts` | 50 | 50 | 3-8 | Done |
| `e2e/specs/setup/wizard.spec.ts` | 52 | 52 | 3-9 / 12-18 | Done |
| `e2e/specs/external-collaborator/accept-link-flow.spec.ts` | 18 | 18 | 9-* | Done |
| `e2e/specs/billing-checkout.spec.ts` | 11 | ~9 | 8-6 | Done |
| `e2e/specs/billing-vat.spec.ts` | 5 | 5 | 8-10 | Done |
| `e2e/specs/trust/public-access.spec.ts` | 34 | 17 | trust epic | Done |
| `e2e/specs/proposals/proposals-workspace.spec.ts` | 39 | 22 | 7-11 / 7-12 | Done |
| `e2e/proposal_generation.spec.ts` | 2 | 2 | 7-5 | Done |

**~240+ tests waiting; ~210 (auth-pages + wizard alone) account for the bulk.**

### Approach (per spec, in order)

1. Inspect each `test.skip(...)` block. Most carry "Activation: remove `test.skip` after Story X" — that story is done.
2. Open the corresponding page in a running dev server (`pnpm dev` from `frontend/`).
3. Run the un-skipped test in headed mode (`--headed --project=chromium`). One of:
   - **Pass:** un-skip permanently, drop the activation comment.
   - **Selectors stale:** repair via `data-testid` or accessible role+name selectors (per `selector-resilience` knowledge).
   - **Backend-dependent state missing:** add a fixture that seeds via API (per `data-factories` knowledge).
   - **Flow no longer matches:** mark `test.fixme(...)` with a one-line reason (`fixme: workspace switcher reordered in S14-3 — re-author after design lock`) and surface to Deb.
4. Re-run all touched tests sequentially three times (mini burn-in per `ci-burn-in` knowledge) to catch flakes before commit.

### Sequencing within P2 (one batch per run, smallest first)

1. **Run P2-a:** `proposal_generation.spec.ts` (2) + `billing-vat.spec.ts` (5) — 7 tests, low-risk warmup.
2. **Run P2-b:** `billing-checkout.spec.ts` (~9) + `accept-link-flow.spec.ts` (18) — 27 tests.
3. **Run P2-c:** `proposals-workspace.spec.ts` (22) + `trust/public-access.spec.ts` (17) — 39 tests.
4. **Run P2-d:** `auth-pages.spec.ts` (50) — auth flows are tight, isolate.
5. **Run P2-e:** `wizard.spec.ts` (52) — onboarding is the largest single surface, save for last.

**Estimated: 5 sub-runs of bmad-testarch-automate / 1 per sub-run.**

### Output per sub-run

- `automation-summary-cluster-a-batch-{a..e}.md` under `eusolicit-docs/test-artifacts/`.
- Per-spec PASS/FIXME/REPAIRED matrix.
- Burn-in report (3 sequential runs, all green).

---

## P3 — New E2E for uncovered done features (cluster C)

**Goal:** Author E2E specs for user-facing features that currently have **no**
browser-level coverage. One feature area per run.

### Coverage map

| Feature area | Page route(s) | Anchor stories | Existing tests |
|---|---|---|---|
| **Tasks / Kanban board** | `/workspace/[id]/tasks` | 10-5, 10-6, 10-14, 10-15 | API tests in client-api; **0 E2E** |
| **Approvals** | `/workspace/[id]/proposals/[pid]/approvals` | 10-8, 10-9, 10-16 | API + service-layer; **0 E2E** |
| **Analytics — ROI** | `/workspace/[id]/analytics/roi` | 12-4 | API isolation only; **0 user E2E** |
| **Analytics — Market** | `/workspace/[id]/analytics/market` | 12-2, 12-3 | API isolation only |
| **Analytics — Team** | `/workspace/[id]/analytics/team` | 12-5 | API isolation only |
| **Analytics — Competitors** | `/workspace/[id]/analytics/competitors` | 12-6 | API isolation only |
| **Analytics — Pipeline** | `/workspace/[id]/analytics/pipeline` | 12-7 | API isolation only |
| **Analytics — Usage** | `/workspace/[id]/analytics/usage` | 12-8 | API isolation only |
| **Reports** | `/workspace/[id]/reports`, `/settings/reports` | 12-9, 12-10 | service-layer tests; **0 E2E** |
| **Calendar settings** | `/workspace/[id]/settings/calendar` | 9-7, 9-8, 9-9, 9-13 | API tests; **0 E2E** |
| **Bid outcomes / lessons** | (workspace bid outcomes pages) | 10-11 | API + integration; **0 E2E** |
| **Comments sidebar** | (proposal workspace) | 10-13 | API tests; **0 E2E** |
| **Content blocks library** | (proposal editor surface) | 7-9, 7-16 | API tests; **0 E2E** |
| **Admin UI** | admin app pages (16 pages) | 12-14 | only `admin-access-control.api.spec.ts` |

### Approach (per feature area)

For each feature area:

1. Spin up `playwright-cli` snapshot of the live page (`tea-browser-automation: auto`) to inventory testable elements.
2. Author **3–6 specs covering the user journeys**:
   - Happy path (read + primary action).
   - Empty state.
   - Error/permission denied.
   - Cross-tenant isolation (where applicable).
3. Use the `<QueryGuard>` and Zustand store patterns documented in CLAUDE.md.
4. Network-first: intercept API calls before navigation; verify with `network-error-monitor` knowledge fragment.
5. Burn-in 3× before commit.

### Sequencing (one feature per run)

Recommended order (highest user-facing risk → lowest):

1. **P3-a:** Tasks/Kanban (highest daily-active-users surface).
2. **P3-b:** Approvals (compliance-critical workflow).
3. **P3-c:** Analytics — start with **Pipeline + ROI** (Professional-tier gated → tier-gate visual coverage).
4. **P3-d:** Analytics — Market + Team + Competitors + Usage (4 remaining dashboards in one run; share a tier-gated fixture).
5. **P3-e:** Reports (PDF/DOCX download flow; cross-cuts proposals).
6. **P3-f:** Calendar settings + Bid outcomes (settings surfaces, share fixture).
7. **P3-g:** Comments + Content blocks (proposal-editor surfaces, share fixture).
8. **P3-h:** Admin UI (separate base URL, separate auth — own run).

**Estimated: 8 sub-runs.**

### Output per sub-run

- New `e2e/specs/<feature>/...spec.ts` files.
- `automation-summary-cluster-c-<feature>.md` per run.
- Updated traceability matrix (re-run `/bmad:tea:trace` for the affected epic).

---

## P4 — API gap fill (cluster D)

**Goal:** Add request-level pytest integration coverage for endpoint modules
where only unit-level / service-layer coverage exists today.

### Targets

| Module | Current state | Gap |
|---|---|---|
| `services/client-api/src/client_api/api/v1/nps_feedback.py` | only `tests/unit/test_nps_feedback_source_inspection.py` | No request/response API test for the actual FastAPI routes — POST feedback, GET status, etc. |
| `services/client-api/src/client_api/api/v1/billing.py` | service-layer (Stripe webhook, VAT, portal, addon checkout) covered in unit; no request-level test for billing routes | API-level happy path + 401/403 + tier-gate negative |
| Cross-service notification flow | `services/notification/tests/...` exists; `tests/cross_service/...` may have partial coverage | End-to-end Redis Streams flow: client-api emits event → notification consumer → outbox row + delivery side effect (mocked SendGrid) |

### Approach

For each:

1. Inventory endpoints (`grep -nE "^@router\." …`).
2. For each endpoint, add an API integration test using the existing
   `client_api_session_factory` + ASGI transport pattern (matches Story 11-7 fixtures).
3. For cross-service: extend `tests/cross_service/` with a test that:
   - Triggers a client-api action that publishes to Redis Streams.
   - Polls the notification service consumer / DB outbox.
   - Asserts side-effect via mocked external delivery (SendGrid stub).
4. Use `recurse` polling fragment for eventual-consistency assertions.

### Sequencing

1. **P4-a:** `nps_feedback.py` API tests (smallest, isolated).
2. **P4-b:** `billing.py` route-level tests (Stripe-mocked, `respx` similar to Story 11-7).
3. **P4-c:** Cross-service notification flow (largest — needs `make up` not just `make infra`).

**Estimated: 3 sub-runs.**

---

## Cross-cutting policies (apply to every phase)

- **No new mocks of client-api endpoints from within client-api tests.** All HTTP calls go through `httpx.AsyncClient` + ASGI transport. External services (AI gateway, Stripe, SendGrid) are mocked with `respx`.
- **Per-test transactional rollback** via `db_session` / per-test redis flush — no test commits.
- **Burn-in:** 3 sequential runs green before commit; track in `automation-summary-*.md`.
- **No `--no-verify` commits**, no skipping pre-commit hooks.
- **Memory:** auto-sync commits routinely break builds (`memory/project_auto_sync_quality.md`); when running these phases, expect to triage failures introduced by recent auto-sync churn before adding new tests.
- **sprint-status.yaml is orchestrator-managed** — do not let any sub-run regenerate it; surgical edits only.

---

## Tracking

Each sub-run updates a small status block here. Format:

```
- [x] P2-a — proposal_generation + billing-vat — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-a-batch-a.md` (1 fixme + 5 un-skipped; CI runtime pending)
- [x] P2-b — billing-checkout + accept-link-flow — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-a-batch-b.md` (20 un-skipped, 7 fixme; CI runtime pending)
- [x] P2-c — proposals-workspace + trust/public-access — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-a-batch-c.md` (27 un-skipped, 6 fixme; CI runtime pending)
- [x] P2-d — auth-pages — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-a-batch-d.md` (48 un-skipped, 0 fixme; CI runtime pending)
- [x] P2-e — wizard — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-a-batch-e.md` (50 un-skipped, 0 fixme; CI runtime pending). **Cluster A closed: 150 active + 14 fixme across 7 spec files.**
- [x] P3-a — Tasks/Kanban — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-c-tasks.md` (5 net-new active tests; CI runtime pending). Drag-reorder + DAG-cycle scenarios deferred.
- [x] P3-b — Approvals — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-c-approvals.md` (5 net-new active tests; CI runtime pending). Pinned-workflow-deleted, fully-approved banner, picker banner, and reject/return variants deferred.
- [x] P3-c — Analytics: Pipeline + ROI — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-c-analytics-pipeline-roi.md` (5 net-new active tests; CI runtime pending). Note: ROI has no UI tier-gate (server-side only) — flagged as a product gap, not a test gap.
- [x] P3-d — Analytics: Market + Team + Competitors + Usage — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-c-analytics-rest.md` (8 net-new active tests; CI runtime pending). Tier-gate visual coverage in this batch is Competitors-only (only one of the four with a UI 403 surface).
- [x] P3-e — Reports — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-c-reports.md` (5 net-new active tests; CI runtime pending). Polling transition, toggle/delete interactions, and Generate Report modal flow deferred.
- [x] P3-f — Calendar + Bid outcomes — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-c-calendar-bid-outcomes.md` (6 net-new active tests across 2 spec files; CI runtime pending). iCal generate POST, OAuth disconnect captures, bid-outcome submit happy path, lessons polling, and server-error 409/422 deferred.
- [x] P3-g — Comments + Content blocks — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-c-comments-content-blocks.md` (4 net-new active tests in `proposals/comments-content-blocks.spec.ts`; CI runtime pending). Populated comment list, resolve/unresolve PATCH captures, search debounce, and Tiptap insert assertion deferred.
- [x] P3-h — Admin UI — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-c-admin-ui.md` (4 net-new active tests in `admin/admin-ui.admin.spec.ts`; CI runtime pending). Tenants list (stub data) + RBAC redirect + pricing-tiers empty/populated. Mutations (create/edit/delete), tenant detail drawer, and other admin surfaces (crawlers, audit-log, analytics, compliance) deferred.
- [x] P4-a — nps_feedback API — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-d-nps-feedback.md` (8 net-new active route-contract tests in `test_nps_feedback_route_contract.py`; CI runtime pending). Bucket/routing computation, score boundaries, validation 422s, cache headers, and unauth gate. PATCH contract + dedup-cache-headers deferred.
- [x] P4-b — billing API — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-d-billing.md` (9 net-new active route-contract tests in `services/client-api/tests/api/test_billing_route_contract.py`; CI runtime pending). Covers `/tiers`, `/subscription`, `/checkout/session`, `/portal/session`, `/invoices`. Webhook/usage RED→GREEN flip + VAT + addon endpoints deferred.
- [x] P4-c — Cross-service notification flow — owner: agent run — date: 2026-05-09 — see `automation-summary-cluster-d-notification-flow.md` (5 net-new active contract tests in `tests/cross_service/test_notification_event_flow.py`; CI runtime pending). Producer envelope shape, TrialExpiring + BidOutcomeRecorded round-trip, correlation_id propagation, producer/consumer stream-name alignment. Uses fakeredis (no testcontainers needed). Outbox + DLQ + true end-to-end with consumer.process_event deferred.
```

Total: **5 (P2) + 8 (P3) + 3 (P4) = 16 sub-runs**.
