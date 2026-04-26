# EU Solicit — Gap Coverage Matrix

**Date**: 2026-04-22 | **Author**: Mary (Business Analyst) | **Status**: Working document
**Purpose**: Audit what was *actually delivered* for each of the 15 gaps identified in `architecture-vs-requirements-gap-analysis.md` (2026-04-05), so that the exhaustive requirements pass (Requirements Brief v5, PRD v2) can identify *additional items* that need new stories to move the gap analysis into delivery.

**Legend**:
- ✅ **Delivered** — gap fully closed in architecture, PRD, and stories
- ⚠️ **Partial** — gap closed at architecture/PRD level, but story scope misses recommendations from the gap doc; additional stories likely needed
- 📝 **Docs-only** — gap is documentation/reconciliation; no code needed; verify doc landed
- ❓ **Verify during exhaustive pass** — story coverage looks complete but exhaustive requirements work may surface sub-scope gaps

---

## P1 Gaps

### GAP-01 — Task Orchestration & Workflow Engine ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §1.12 (ADR), §13.2 Task Orchestration Architecture |
| PRD v1 feature row | §3.4 "Task orchestration" (one-liner — needs expansion) |
| Data model | `client.tasks`, `client.task_dependencies`, `client.task_templates` (migration confirmed) |
| Stories | 10-5 (CRUD), 10-6 (DAG cycle rejection), 10-7 (templates application), 10-14 (Kanban UI), 10-15 (template manager UI) |
| Events | Task lifecycle events via Notification Service (9-10) |

**Exhaustive-pass watchlist** — confirm: task recurrence rules, subtasks, time-zone handling for due dates, bulk operations, attachment model on tasks, task template parameterization per opportunity type (procurement vs. grant), overdue notification cadence.

---

### GAP-02 — Approval Gates ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §1.12 (co-located with GAP-01 ADR) |
| PRD v1 feature row | §3.4 "Approval workflows — Enterprise" |
| Data model | `client.approval_workflows`, `client.approval_stages`, `client.approval_decisions` |
| Stories | 10-8 (workflow/stages CRUD), 10-9 (decision engine incl. `auto_advance`, `approval.decided` event), 10-16 (pipeline UI) |
| Events | `approval.requested`, `approval.decided` (Redis Streams) |

**Exhaustive-pass watchlist**: delegation/substitute reviewers, parallel vs. serial stage execution semantics, SLA tracking on pending approvals, approval policy change impact on in-flight proposals, audit of approver identity at time of decision.

---

### GAP-06 — Guided Submission Workflows ⚠️ PARTIAL

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §1.15 (ADR self-service bidding) |
| PRD v1 feature row | §3.1 "Submission guides — Paid" (one-liner) |
| Data model | `pipeline.submission_guides` (auto-generated, `reviewed=false` flag) |
| Stories | 5-8 (AI generation task), 6-5 / 6-11 (display in opportunity detail) |

**GAPS IN DELIVERY** (additional stories required):
1. **No admin CRUD for submission guide templates** — gap doc explicitly said "Admin API should manage guide templates." Currently guides are purely AI-generated per opportunity; no per-portal template library that admins curate.
2. **No admin review/approve flow** — the `reviewed` flag exists on rows but there is no UI/API to flip it. All guides ship to users unreviewed.
3. **No user-facing step-by-step checklist with per-step acknowledgement** — displayed as reference content on opportunity detail only; no interactive completion tracking, no "I submitted step N" state.
4. **No portal-specific instruction schema / versioning** — a portal's submission process may change; need schema for guide versions tied to portal metadata.

**Proposed new stories** (to be sequenced by orchestrator):
- **S13.1** — Admin API: Submission Guide Template CRUD (per source/portal)
- **S13.2** — Admin API + UI: AI-generated guide review queue (flip `reviewed=true`, edit/override, publish)
- **S13.3** — Client UI: Interactive submission checklist with per-step acknowledgement + persistence
- **S13.4** — Portal metadata & guide versioning schema migration

---

### GAP-07 — 14-Day Free Trial ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §1.14, §10.1 Trial Management |
| PRD v1 feature row | §3.9 "14-day trial" |
| Data model | `subscriptions.trial_start`, `trial_end`, `trial_converted_at`; Stripe `trial_end` on subscription object |
| Stories | 8-3 (provisioning), 8-5 (expiry/downgrade logic), 8-13 (trial banner UI), 9-11 (Stripe usage sync on expiry) |
| Notifications | `customer.subscription.trial_will_end` → reminder email (9-6 / 9-11) |

**Exhaustive-pass watchlist**: abuse prevention (one trial per company? per email domain? per payment method?); trial-to-paid conversion during active trial (prorating behavior); Free tier data retention policy post-expiry; trial extension flow (does admin have override?).

---

### GAP-10 — Proposal Collaboration Model ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §1.13 ADR; §13.1 Proposal Collaboration Architecture |
| PRD v1 feature row | §3.4 "Per-proposal roles", "Section locking", "Proposal comments" |
| Data model | `client.proposal_collaborators`, `client.proposal_section_locks`, `client.proposal_comments` |
| Stories | 10-1 (collaborator CRUD), 10-3 (section locking 15min TTL, 423 on conflict), 10-4 (comments anchored to section+version), 10-12 (collaborator/lock UI), 10-13 (comments sidebar UI) |

**Exhaustive-pass watchlist**: @mention → notification dispatch, comment resolution + carry-forward across versions (ATDD says covered — verify), concurrent-edit UX for unlocked sections, lock-override policy for admins, audit of who held lock & for how long, lock release on user logout / session expiry.

---

### GAP-11 — Per-Tender RBAC Granularity ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §1.13 (co-located with GAP-10) |
| PRD v1 feature row | §3.4 "Entity-level RBAC" |
| Data model | `client.entity_permissions` (user_id, company_id, entity_type, entity_id, permission) with UNIQUE constraint |
| Stories | 2-10 (entity-level RBAC middleware), 10-2 (proposal-level RBAC enforcement) |
| Rules | Company role ceiling + entity permission ≤ ceiling; admin/bid_manager bypass for own company only |

**Exhaustive-pass watchlist**: per-opportunity permission grants (beyond per-proposal) — the gap doc mentioned "opportunity_access" pair — is this needed? Bulk grant revocation on user deactivation or company leave. Inherited permissions (assigning a template grants all its permissions). Cross-company consortium scenarios (user in company A edits shared proposal in company B's workspace).

---

## P2 Gaps

### GAP-05 — ESPD Auto-Fill ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §1.15, §5 Data Model (`espd_profiles` ★NEW), §10 |
| PRD v1 feature row | §3.6 "ESPD auto-fill — Professional+" |
| Data model | `client.espd_profiles` (JSONB `espd_data` mapped to ESPD schema sections) |
| Stories | 2-12 (baseline CRUD), 11-1 (full schema migration), 11-2 (profile CRUD API), 11-3 (auto-fill agent), 11-13 (management frontend) |

**Exhaustive-pass watchlist**: ESPD XML export format + schematron validation, multi-entity profile switching within a single bid (consortium arrangements), ESPD part IV–VI granularity, AI-assist for free-text fields, version history on profiles.

---

### GAP-08 — Per-Bid Add-On Pricing ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §1.14, §10.2 |
| PRD v1 feature row | §3.9 "Per-bid add-ons" |
| Data model | `client.add_on_purchases` (scoped by opportunity_id) |
| Stories | 8-9 (purchase flow via Stripe Checkout) |

**Exhaustive-pass watchlist**: add-on refund policy, bundle pricing (e.g., full-proposal add-on includes compliance add-on), add-on expiry (time-limited?), add-on gifting/transfer between collaborators on same bid, invoice/receipt issuance for one-time charges.

---

### GAP-09 — EU VAT Handling ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §10.3 EU VAT Handling |
| PRD v1 feature row | §3.9 "EU VAT" |
| Data model | `companies.tax_id`; Stripe `customer.tax_ids` sync |
| Stories | 8-10 (Stripe Tax + VIES validation), 8-11 (enterprise custom invoicing) |
| Behaviour | Reverse charge per Article 196 Council Directive 2006/112/EC |

**Exhaustive-pass watchlist**: non-EU customer taxation, UK post-Brexit VAT handling, tax ID update mid-cycle (reverse charge retroactive?), VAT exemption flags (e.g., non-profit municipalities), credit notes on refunds.

---

### GAP-12 — Reusable Content Blocks Library ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §5 Data Model, §6 Service modules |
| PRD v1 feature row | §3.3 "Content blocks library" |
| Data model | `client.content_blocks` (tagged, versioned, approved) |
| Stories | 7-9 (CRUD + search API), 7-16 (library UI + insert-into-proposal) |
| RAG | Content blocks fed into Proposal Generator Workflow via KraftData Company Profiles Store |

**Exhaustive-pass watchlist**: block approval workflow (who can mark "approved"?), usage analytics per block (most-inserted), block inheritance (parent blocks with variant overrides), block lifecycle (archive/deprecate), translation alignment (BG/EN block pairs).

---

### GAP-13 — Bid History Analytics & Custom Reporting ✅

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §13.3 Analytics Computation Model (materialized views refreshed by Notification Service Celery Beat) |
| PRD v1 feature row | §3.8 all analytics features |
| Data model | `client.bid_outcomes` (+ Migration 045 runtime columns), `client.bid_preparation_logs` (time + cost per user per proposal), `mv_roi_tracker`, `mv_team_performance`, other MVs |
| Stories | 10-11 (outcome + prep-logs runtime + Lessons Learned agent), 12-1 (MV infrastructure), 12-2/3 (market intel dashboard), 12-4 (ROI tracker), 12-5 (team performance), 12-6 (competitor intel), 12-7 (pipeline forecast), 12-8 (usage dashboard), 12-9 (report generator PDF/DOCX), 12-10 (scheduled/on-demand report delivery), 9-14 (scheduled report Celery task) |
| Agents | Bid History Analytics Agent, Lessons Learned Agent |

**Exhaustive-pass watchlist**: custom report builder (user-defined metric combinations), export scheduling granularity (cron-style?), dashboard drill-down to source rows, GDPR export filters for team-performance metrics, benchmark comparisons across companies (anonymized).

---

### GAP-14 — KraftData Agent Inventory Reconciliation 📝

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §11.2 Agent Inventory (29 entries; reconciled) |
| PRD v1 | Partially reflects agent coverage in §3.1–§3.8 |
| Requirements Brief v4 | **Still lists 22 entries, unreconciled** — this is the exact reason for v5 |

**Outstanding action**: Update Requirements Brief §7 (Pre-Built Agent Inventory) to match Architecture v4 §11.2 — 29 agents including Consortium Finder, Reporting Template Generator, ESPD Auto-Fill, Submission Guide, Lessons Learned.

---

### GAP-15 — Crawler Implementation Strategy Documentation 📝

| Dimension | Evidence |
|-----------|----------|
| Architecture v4 | Crawlers implemented as local Celery tasks in `data-pipeline` service (5-4, 5-5, 5-6). |
| Requirements Brief v4 | Still describes crawlers as KraftData agents (contradicts built implementation) |

**Outstanding action**: Requirements Brief v5 §5.2 and §7 must explicitly document: "Crawlers are deterministic HTTP scrapers implemented as local Celery tasks (AOP, TED, EU Grants). KraftData agents are used only for normalization/enrichment downstream of crawling." Remove crawler agents from §7's KraftData inventory.

---

## P3 Gaps

### GAP-03 — Consortium Finder ✅ (delivered despite "deferrable" original rating)

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §11.2 Agent 14; Partner organization storage |
| PRD v1 feature row | §3.5 "Consortium finder — Professional+" |
| Stories | 11-6 (agent integration), 11-12 (frontend panel) |

**Exhaustive-pass watchlist**: partner opt-in/opt-out of discoverability, privacy/GDPR for individuals vs. organizations, partner match confidence scoring, direct-message/contact initiation flow (does the platform broker the introduction or just surface contacts?), consortium history tracking for repeat partnerships.

---

### GAP-04 — Reporting Template Generator (Post-Award) ⚠️ PARTIAL

| Dimension | Evidence |
|-----------|----------|
| Architecture closure | §11.2 Agent 15; Phase 5 roadmap |
| PRD v1 feature row | §3.5 "Reporting template generator — Enterprise" |
| Stories | 11-7 (logframe + reporting template agent integrations — status: review / marked done) |

**GAPS IN DELIVERY** (the gap doc explicitly said "bid_outcomes table needs expansion to track ongoing project data: milestones, reporting periods, financial draws"):

Current `bid_outcomes` tracks *submission outcome* (won/lost/withdrawn) + lessons learned. There is **no post-award project data model**:
- No `awarded_projects` / `project_milestones` table
- No `reporting_periods` schedule model (e.g., quarterly technical reports, annual financial reports)
- No `financial_draws` / budget-consumption tracking
- No periodic report submission workflow (draft → review → submit to programme authority)

Story 11-7 generates report *templates* from a narrative — it does not manage an ongoing awarded project.

**Proposed new stories** (new mini-epic, roughly Epic 13 "Post-Award Project Management"):
- **S13.5** — Schema: `awarded_projects`, `project_milestones`, `reporting_periods`, `financial_draws`
- **S13.6** — Client API: Awarded Project CRUD + auto-provision on `bid_outcome = won`
- **S13.7** — Reporting period scheduler (auto-generates periodic report stubs on schedule)
- **S13.8** — Financial draw ledger API (budget vs. actual tracking)
- **S13.9** — Enterprise frontend: Awarded Projects dashboard with reporting calendar

**Alternative (recommended for scope control)**: Park these as P3 forever and keep GAP-04 as "template generation only (story 11-7)" — explicitly out-of-MVP for post-award management. Update PRD §7 Out-of-Scope accordingly.

---

## Summary

| Gap | Priority | Status | Additional Items Needed |
|-----|----------|--------|--------------------------|
| GAP-01 Task Orchestration | P1 | ✅ Delivered | Confirm during exhaustive pass |
| GAP-02 Approval Gates | P1 | ✅ Delivered | Confirm during exhaustive pass |
| **GAP-06 Submission Guides** | **P1** | **⚠️ Partial** | **4 new stories (S13.1–S13.4)** |
| GAP-07 14-Day Trial | P1 | ✅ Delivered | Confirm during exhaustive pass |
| GAP-10 Proposal Collaboration | P1 | ✅ Delivered | Confirm during exhaustive pass |
| GAP-11 Per-Tender RBAC | P2 | ✅ Delivered | Confirm during exhaustive pass |
| GAP-05 ESPD Auto-Fill | P2 | ✅ Delivered | Confirm during exhaustive pass |
| GAP-08 Per-Bid Add-On | P2 | ✅ Delivered | Confirm during exhaustive pass |
| GAP-09 EU VAT | P2 | ✅ Delivered | Confirm during exhaustive pass |
| GAP-12 Content Blocks | P2 | ✅ Delivered | Confirm during exhaustive pass |
| GAP-13 Analytics & Reporting | P2 | ✅ Delivered | Confirm during exhaustive pass |
| **GAP-14 Agent Inventory** | P2 | 📝 Docs-only | **Update Req Brief §7 (29 agents)** |
| **GAP-15 Crawler Strategy** | P2 | 📝 Docs-only | **Update Req Brief §5.2 + §7** |
| GAP-03 Consortium Finder | P3 | ✅ Delivered | Confirm during exhaustive pass |
| **GAP-04 Post-Award Reporting** | **P3** | **⚠️ Partial** | **Decision: defer as out-of-MVP, or 5 new stories (S13.5–S13.9)** |

**Total confirmed additional items**: 2 doc-only updates + 4 stories for GAP-06 + {0 or 5} stories for GAP-04 depending on scope decision.

---

## Open decisions for Deb

1. **GAP-06 Submission Guides** — confirm the 4 proposed stories (S13.1–S13.4) are in-scope. Without them, submission guides are AI-generated and shipped to users unreviewed, and there's no curated portal template library.
2. **GAP-04 Post-Award** — *new mini-epic* (S13.5–S13.9) OR *explicit out-of-MVP declaration* in PRD v2? The gap doc originally said "can be deferred," but if we don't deliver it we need to say so clearly in PRD §7.
3. **Exhaustive-pass depth** — how deep on "Confirm during exhaustive pass" rows? Every watchlist item becomes requirements text in PRD v2, but not every watchlist item becomes a *new story*. If we surface a sub-scope gap during exhaustive writing (e.g., "task recurrence isn't implemented"), should that also become a proposed new story, or is the exhaustive pass purely descriptive of what's built?
