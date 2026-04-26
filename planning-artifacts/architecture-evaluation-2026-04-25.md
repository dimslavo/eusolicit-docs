---
documentType: 'architecture-evaluation'
sourceAmendment: 'eusolicit-docs/planning-artifacts/prd-amendment-2026-04-25.md'
sourceResearch: 'eusolicit-docs/planning-artifacts/research/market-eu-govwin-tender-intelligence-research-2026-04-25.md'
basePRD: 'eusolicit-docs/planning-artifacts/PRD.md (v1.1)'
baseArchitecture: 'eusolicit-docs/EU_Solicit_Solution_Architecture_v5.md (v5.0)'
baseDecomposition: 'eusolicit-docs/EU_Solicit_Service_Decomposition_Analysis.md (v1.1)'
date: '2026-04-25'
author: 'Winston (System Architect)'
status: 'Approved — Deb signed off on all open questions 2026-04-25; ready for sprint-change-proposal'
nextSkills: ['bmad-correct-course', 'bmad-create-epics-and-stories']
---

# Architecture Evaluation 2026-04-25 — PRD Amendment "Tenders + Grants for EU Consulting Firms"

## What this document is

A technical-feasibility evaluation of the six PRD-level changes Mary proposed in `prd-amendment-2026-04-25.md`. For each change I give you **a verdict** (feasible-as-stated / feasible-with-modifications / blocked-needs-redesign), **the trade-offs** I considered, and **the concrete modifications** I'd ship. Where Mary's amendment is already aligned with the v5 Solution Architecture I say so plainly — three of these changes are smaller architecturally than the brief implies; one is materially larger.

The artefact is sequenced to flow into `bmad-correct-course` (sprint-change-proposal) and then `bmad-create-epics-and-stories` without re-derivation. If you only have time for one section, read **§7 Refined Epic Sequencing** and **§9 Open Questions Blocking Implementation**.

---

## 1. Top-line architectural verdict

> **The amendment is feasible. None of the six changes require redesigning the v5 architecture. Five of six are extensions of patterns already shipped in Epics 1–13. The exception is the multi-client workspace data model — that one is a tenancy-model refactor that touches schema, RBAC, audit trail, billing, and the entire frontend route tree, and I'd argue it deserves more sub-stories than the amendment implies.**

The amendment also under-states two things I want surfaced before epic decomposition:

1. **The 99.9% SLA is a platform-engineering programme, not a documentation change.** PostgreSQL and Redis are single-instance today. We can't credibly publish 99.9% until they aren't, plus we close the k6 baseline carry-forward that's been deferred 6 epics running.
2. **Per-bid SKU is already in v5** — `add_on_purchases` table (line 584), Stripe one-time payment flow (line 741), ADR §1.14. Mary's amendment is *renaming and re-pricing* the existing add-on, not adding a new payment primitive. Worth knowing before estimating.

Taken together: this is roughly **+2 quarters of platform engineering** plus **+1 net-new service** plus **a coordinated tenancy migration**. None of that is alarming for a project that has already shipped 13 epics — but the *sequencing* matters more than the *scope*, which is what most of this evaluation is about.

---

## 2. Per-change evaluation

### Change 1 — Multi-Client Workspace Data Model

**Verdict: 🟡 Feasible with modifications. Larger than implied — recommend Epic 14 splits into a 5-story arc.**

**Trade-offs considered**

The amendment proposes two model shapes (new entity above `company`, or `company.parent_company_id`). Neither is right. I'd argue for **a third option**:

- **Keep `company` as the tenant boundary** (no rename — the BMAD code, RBAC rules, billing schema all assume `company_id` as the root key, and renaming is a 13-epic regression risk).
- **Introduce a new `client_workspace` entity that is *child of* company.** A consulting-firm tenant is a `company`; each engagement is a `client_workspace` row in `client.client_workspaces`.
- **Existing single-tenant accounts are auto-backfilled with one default workspace named "Default"** at migration time. New accounts get one default workspace at registration. The single-workspace case stays UX-invisible until the user creates a second workspace.

This avoids: (a) renaming `company` (which would touch ~50% of the codebase per a quick grep of the existing services), (b) introducing a new "tenant" concept that competes with existing language in the project. Instead we **promote `company` to mean "tenant" implicitly** by adding workspaces underneath it. The Rule of Three says: don't abstract until the third instance exists. Workspaces are the third instance — first was company, second was opportunity-scoped entity permissions.

**Schema-isolation impact:** None. Workspaces are row-level scoping inside the `client` schema — the per-service schema isolation pattern is preserved verbatim. **No PostgreSQL Row-Level Security (RLS) required for correctness** — the existing `check_entity_access()` factory plus a new `WorkspaceScope` Depends() in FastAPI is enough.

**RLS as defence-in-depth for ISO 27001:** Worth considering as an audit-evidence multiplier, but **not blocking**. Decision deferred to ISO 27001 audit prep (Epic 18 / parallel programme).

**RBAC extension is genuinely small:** `check_entity_access()` already takes scope; we add `workspace_id` as an additional scope dimension. Plus a tenant-level `tenant_admin` role for cross-workspace access. No rewrite, no new factory.

**Audit trail:** Add a nullable `workspace_id` column to `shared.audit_log`, default NULL for tenant-level mutations (e.g., billing). Backfill with workspace-id-or-null based on entity type. Migration is non-destructive.

**External collaborator auth:** Magic-link with scoped JWT, 30-day expiry — confirmed. Stripe seat consumption opt-out via a new `external_collaborator` claim that excludes the user from active-seat counts. Aligned with existing JWT RS256 pattern (Epic 2).

**Concrete modification — Epic 14 sub-story arc (5 stories, not 1)**

| # | Story | Why discrete |
|---|---|---|
| **S14.00** | Schema migration + atomic backfill of single-default-workspace for existing companies | Migration must complete in one transaction with rollback safety; cannot be coupled to feature work |
| **S14.01** | Workspace CRUD (create / rename / archive / delete) at tenant-admin level + audit trail | API + persistence; standalone deliverable |
| **S14.02** | RBAC extension: workspace-scoped roles, `WorkspaceScope` Depends() factory, cross-workspace `tenant_admin` bypass | Security-critical; must ship before any workspace-scoped endpoint goes live |
| **S14.03** | Frontend workspace switcher + URL routing `/[locale]/workspace/[workspaceId]/...` + Zustand persist namespace migration | Significant frontend change; must ship after S14.01/S14.02 to have backend to talk to |
| **S14.04** | External collaborator magic-link flow + comment-only RBAC role + audit trail flagging | Independent of S14.03; can ship in parallel |

**Frontend impact (significant — call this out explicitly):** Every protected page in `apps/client/app/[locale]/(protected)/` gets a new path segment. The Zustand persist key changes from `eusolicit-client-auth-store` to `eusolicit-client-auth-store-v2` (per-workspace state). Existing customers' localStorage entries from v1 are gracefully migrated on next load (read v1 → write v2 → delete v1; no data loss). This is the single biggest frontend test-debt risk in the amendment — recommend allocating 2 ATDD-heavy sprints for S14.03.

**One thing the amendment didn't address:** **Cross-workspace content-block library.** A consulting firm wants to maintain shared boilerplate (company overview, sustainability policy) across all client engagements. Recommend introducing a `tenant_content_block` table at tenant scope that workspaces can import-by-reference, and per-workspace overrides for client-specific edits. Add as S14.05 or carry into Epic 19.

---

### Change 2 — Per-Bid Pricing SKU

**Verdict: 🟢 Feasible as stated. *Already partially scoped* in v5 architecture — this is a UX/pricing refinement, not net-new architecture.**

**What's already shipped in v5**

- `add_on_purchases` table in `client` schema (Sol Arch v5 §8, line 584)
- ADR §1.14 — "per-bid add-on purchases as Stripe one-time charges (`mode: payment`)"
- Solution Architecture v5 §13.x — flow described: "User selects add-on from opportunity detail → Stripe Checkout Session → on `checkout.session.completed` webhook, create `add_on_purchases` record with `status: succeeded` → unlock the feature for that specific opportunity"
- Stripe billing patterns from Epic 8 — dual-layer idempotency, ECDSA webhook validation, `tier_cache` with DELETE-on-change

**What's net-new from the amendment**

- **Pricing tiers**: €99 / €199 / €299 by feature stack. This is a pricing-config change; Stripe Price IDs in `infra/stripe-config.yaml` and a small `add_on_pricing_tiers` table.
- **Pro+ tier**: A new tier between Professional and Enterprise. This adds one row to the existing `tier_access_policies` table and one enum value (`pro_plus`) to the `subscription.tier` column. **Trade-off:** I'd push back gently on creating "Pro+" as a distinct tier vs. "Professional + multi-workspace add-on." A single Pro+ tier is simpler to message and easier to gate; an add-on creates more billing complexity. Recommend **Pro+ as a distinct tier**.
- **Metering bypass**: When `add_on_purchases.opportunity_id = X AND status = succeeded`, the existing `_USAGE_LUA` script (Epic 6 atomic Redis Lua — "GET + conditional INCR + EXPIRE in single script") gets a pre-check: skip the tier cap if an add-on is active for the opportunity in scope. **This is a single Lua-script edit, not a redesign.** The contention concern is unfounded — checks are still single-key atomic.

**Concrete modifications**

- **Refund policy**: Strongly recommend non-refundable per-bid charges with a 24-hour cancellation window (Stripe-standard). Refund flow exists in Epic 8 webhook handling — reuse.
- **Bid identification trigger**: Use the existing `bid_decisions` flow — when `bid_decision.recommendation == 'bid'` is logged, fire an `addon.opportunity_engaged` event. Per Sol Arch v5 §5.2 event catalog this aligns with existing patterns.
- **Renaming**: The amendment calls it "Per-Bid SKU"; v5 calls it "add-on purchase". Suggest aligning UX language to "Per-Bid" (Mary's framing is buyer-facing; v5 was internal). Backend tables/events stay `add_on_*` for migration safety.
- **EU VAT**: Stripe Tax already handles VAT MOSS for one-time charges (per ADR §1.14). One small tweak — confirm Stripe Tax product mapping for "per-bid" SKUs separately from subscription SKUs in tax-reporting reconciliation.

**Epic 15 sub-story arc (3 stories — smaller than implied)**

| # | Story |
|---|---|
| **S15.00** | Pro+ tier definition + `tier_access_policies` row + Stripe Price ID + tier-cache extension |
| **S15.01** | Per-bid pricing-tier configuration (admin-configurable €99/€199/€299) + Stripe Checkout integration + `add_on_pricing_tiers` table |
| **S15.02** | Metering bypass: `_USAGE_LUA` extension + integration tests for pre-check correctness under concurrent INCR |

---

### Change 3 — CRM + Slack/Teams Integrations

**Verdict: 🟡 Feasible with modifications. New service is the right call. Sequence Slack/Teams before CRM.**

**Service location decision**

The amendment asks: new `integrations-api` service, or extend `client-api`?

I'd ship **a new `integrations-api` service**, for four reasons that mirror the AI Gateway split rationale (Sol Arch v5 §1.2, §3.4):

1. **Blast radius:** HubSpot/Salesforce/Pipedrive outages or rate-limit hits should not cascade into client-api request handling. Bid managers searching opportunities don't care that HubSpot is rate-limiting our sync.
2. **Rate-limiting:** Each provider has different limit characteristics (HubSpot: 100/10s, Salesforce: 24h limits, Pipedrive: 100/2s). Centralised circuit-breaker per logical name (Epic 4 pattern) keeps it clean.
3. **Sync workers belong in their own deployment:** Bi-directional sync needs Celery workers + Beat scheduling. Adding to client-api couples web-pod scaling to sync-worker scaling. Notification service is the analogue.
4. **Rule of Three confirmation:** HubSpot + Salesforce + Pipedrive at launch = 3 providers. Slack + Teams = 2 more. Five integrations is past the threshold.

**Trade-off for the record:** If we believed only HubSpot would ever ship (single provider), I'd argue against the new service. Three concurrent providers tips it.

**Sequence: Slack/Teams **before** CRM**

The amendment lists CRM as Epic 16 and Slack/Teams as Epic 17. I'd flip them. Reasoning:

- **Slack/Teams via incoming webhooks is ~5% of the engineering scope of CRM bi-directional sync.** It ships in 1–2 weeks; CRM ships in 6–10 weeks per provider.
- **Slack/Teams demonstrates value to the bid team daily; CRM demonstrates value to the bid director quarterly.** Daily-felt features build adoption faster, which informs the CRM design (we'll know which workflows need CRM hooks).
- **Slack/Teams is Pro tier; CRM is Pro+ tier.** Larger addressable market sooner.

So: rename Mary's Epic 16/17 → Epic 16 = Slack/Teams (Pro tier), Epic 17 = CRM (Pro+ tier).

**Concrete modifications to Epic 17 (CRM)**

- **Sync model:** Webhook-driven CRM→us (real-time on CRM-side changes) + scheduled poll us→CRM every 15 minutes (push on EU Solicit-side changes; Celery Beat). Per-Provider Celery worker queue. **Don't** attempt full event-streaming; CRM webhooks are reliable enough at our scale.
- **OAuth token storage:** Fernet-encrypted (project-context Epic 9 standard) in a new `client.crm_connections` table with workspace scoping. Token rotation supported via Celery Beat task.
- **Conflict resolution:** Last-Write-Wins with a workspace-visible conflict log. Each conflict logged to `shared.audit_log` with `conflict_resolution_strategy='lww'` for ISO 27001 evidence.
- **Provider sequencing within Epic 17:** HubSpot first (research §Step 4 confirms BG/CEE/SME density), then Pipedrive (lower complexity, fast follow), then Salesforce (more complex; 6-week add). Keep this order even if the architect reverses Mary's CRM-priority recommendation, since complexity dictates order.

**Epic 16 (Slack/Teams) — 1-story arc**

| # | Story |
|---|---|
| **S16.00** | `integrations-api` service bootstrap + webhook config UI in client + Slack/Teams incoming-webhook templates + alert routing |

**Epic 17 (CRM) — 4-story arc, one per provider plus shared infra**

| # | Story |
|---|---|
| **S17.00** | CRM connection model + OAuth token vault (Fernet) + sync engine scaffold + conflict-resolution framework |
| **S17.01** | HubSpot adapter (Deals + Contacts) — full bi-directional sync |
| **S17.02** | Pipedrive adapter |
| **S17.03** | Salesforce adapter (rate-limit-aware; daily quota) |

---

### Change 4 — ISO 27001 Roadmap & Trust Center

**Verdict: 🟢 Feasible as stated. The Trust Center page is small engineering. The ISO 27001 audit is a 6–12 month controls programme, not a coding task.**

**Trust Center page architecture**

Static rendering at build-time from version-controlled YAML/MDX is correct. Use the existing Next.js MDX pipeline; no DB-backed CMS.

**Layout:**
- `apps/client/app/(public)/trust/page.tsx` — public route, no auth required (ingress allows `/trust/*` without JWT)
- Source: `apps/client/content/trust/*.mdx` and `apps/client/content/trust/sub-processors.yaml`
- PDF artefacts: stored in `infra/trust/artefacts/` (Git-tracked), served via signed S3 URLs with versioning
- Sub-processor changes detected via Git hook → CI emits `subprocessor.changed` event → `notification` service emails active customer DPAs (existing email pipeline, new template)

**Trade-off:** Static-rendered Trust Center means a code deploy on every change. The amendment suggests "auto-update from a structured source." For weekly cadence this is fine — Git-driven workflow is auditable (which is exactly what ISO 27001 wants for change management). If pace ever exceeds weekly, revisit. Until then, **boring technology beats clever CMS**.

**PDF artefact split (per Mary's recommendation, accepted)**

| Artefact | Source | Method |
|---|---|---|
| GDPR DPA | Legal-curated | Human PDF, version-controlled in `infra/trust/artefacts/dpa/v{N}.pdf` |
| Security Overview | Engineering + Legal | Generated from MDX via WeasyPrint (Epic 7 pipeline) |
| Sub-Processor List | `sub-processors.yaml` | Generated from YAML via WeasyPrint, change-log appended |
| Pen-Test Summary (redacted) | Auditor-curated | Human PDF, version-controlled |
| BCP Summary | Operations | Generated from MDX |
| Data Residency Confirmation | Engineering | Generated from MDX |

**ISO 27001 audit-prep programme — engineering deliverables**

This is the part Mary is right to call out as "sprint-spanning." Most of the controls are *already* in place from existing epics — but the *evidence-collection* burden is real. From a quick mapping:

| ISO 27001 Annex A control | Status | Source epic |
|---|---|---|
| A.5 Information security policies | Need formalisation | New |
| A.6 Organization of information security | Need formalisation | New |
| A.8 Asset management | Partial — sub-processor list only | New |
| A.9 Access control | ✅ Shipped | Epic 1, 2 (RBAC + JWT) |
| A.10 Cryptography | ✅ Shipped | Epic 9 (Fernet for OAuth) |
| A.12 Operations security | Partial | Epic 4 (resilience), Epic 5 (audit) |
| A.13 Communications security | ✅ Shipped | Epic 1 (TLS, schema isolation) |
| A.14 System acquisition / dev / maintenance | ✅ Shipped | Epic 3 (BMAD process), TEA |
| A.16 Information security incident management | Need runbooks | New |
| A.17 Business continuity | Need BCP doc + DR test | New |
| A.18 Compliance | ✅ Shipped (GDPR) + ISO programme | Epic 7, 8, 13 |

**Engineering net-new for ISO 27001:** ~3 stories of evidence-collection + runbook authoring + DR test + access-review automation. Not a quarter; closer to 3–4 sprints in parallel with feature epics.

**ISO audit dependency chain — refined**

Mary said audit is "Q3-Q4 2026 if amendment lands in Q2." I'd refine:

- M0 (now): Amendment approved
- M1: Trust Center page live (Epic 18); ISO 27001 roadmap published with milestones
- M2-3: Controls inventory + gap analysis (in parallel with Epic 14)
- M3-5: Evidence collection + runbook authoring + DR test
- M6: Auditor engaged; Stage 1 audit (documentation review)
- M9-10: Stage 2 audit (controls review)
- M12: Certification awarded

**This is achievable** if it's resourced as a parallel programme with a named engineering lead. Without dedicated ownership, it slips to M18.

**Epic 18 sub-story arc (3 stories)**

| # | Story |
|---|---|
| **S18.00** | Trust Center route + MDX pipeline + sub-processor YAML + change-log generator |
| **S18.01** | PDF artefact pipeline (WeasyPrint reuse from Epic 7) for living artefacts; Git-managed for legal artefacts |
| **S18.02** | Sub-processor change → DPA notification flow (notification service + new template) |

---

### Change 5 — 99.9% Uptime SLA

**Verdict: 🟠 Feasible with modifications. Material infra uplift required before publishing the SLA. Not just an observability change.**

**Where we are today**

The v5 architecture documents single-instance PostgreSQL and Redis. Sol Arch v5 §3 declares 5 services with HPA scaling, but **PDBs and minimum-replica counts are not specified**. The k6 baseline carry-forward in project-context.md is now into its 6th epic — meaning we have **no measured baseline** for current-state availability.

This means three things, in order:

1. We cannot credibly publish 99.9% (43 min/month max downtime) when our control plane has SPoFs.
2. We cannot measure progress without the k6 baseline closure.
3. We need a platform-engineering workstream BEFORE the SLA is customer-facing.

**Concrete infra uplift required**

| Component | Current | Required for 99.9% |
|---|---|---|
| PostgreSQL | Single instance | HA primary + sync replica + automated failover. Recommend managed (RDS Multi-AZ or AWS Aurora) — **boring tech beats Patroni for this team size** |
| Redis | Single instance | Redis Sentinel (or managed Redis Cluster) — minimum 1 primary + 2 replicas |
| All services | HPA configured | Add PodDisruptionBudgets with `minAvailable: 1` and minimum replicas ≥ 2 per service in production |
| Ingress | nginx-ingress | Already 2-replica per typical Helm chart; verify; add PDB if missing |
| KraftData dependency | External | Already isolated via AI Gateway circuit breaker (Epic 4); 99.9% applies to the *EU Solicit platform*, not KraftData. Make the SLA exclude AI-Gateway-dependent endpoints during KraftData incidents |
| Monitoring | Prometheus + Grafana | Add SLO dashboards with error-budget tracking; alert on burn rate |
| On-call | Not formalised | PagerDuty/Opsgenie + runbook density + on-call rotation policy |

**Engineering effort: ~2 sprints of platform-engineering work**, plus ongoing on-call commitment.

**Recommended sequencing**

- **Phase A (M0–M2, parallel to Epic 14):** k6 baseline closure, PostgreSQL HA migration, Redis Sentinel migration. SLO dashboards.
- **Phase B (M3, after Phase A):** PDB rollout, on-call formalisation, runbook authoring.
- **Phase C (M4):** Public SLA announcement with service-credit policy. Trust Center updated.

**Don't publish the SLA before Phase C completes.** Pre-announcing creates legal risk if breached without infrastructure to support it.

**Epic — not Mary's Epic 5/non-epic, but a parallel platform workstream**

| # | Story |
|---|---|
| **PE.01** | Close k6 baseline carry-forward (load tests for client-api, AI-gateway, data-pipeline endpoints) — must close before declaring any SLO |
| **PE.02** | PostgreSQL HA migration (cutover with rollback safety) |
| **PE.03** | Redis HA migration |
| **PE.04** | PDB rollout + min-replica enforcement across all services |
| **PE.05** | SLO dashboards + error-budget alerting |
| **PE.06** | On-call rotation + runbook authoring + incident-management process |

**This is 6 platform-engineering stories — material — and the amendment classified it as "no new epic." I disagree.** Treat as Epic 21: Platform Reliability for 99.9% SLA.

---

### Change 6 — Outcome Telemetry & Renewal Proof Brief

**Verdict: 🟢 Feasible as stated. Stay in client-api with a dedicated module. Volume does not justify a new service.**

**Service location**

Mary's recommendation — extend `client-api` analytics module rather than spin up `analytics-api` — is correct. The Rule of Three reasoning: we have analytics today (FR7.1, materialized views), we'd be adding outcome telemetry (one more analytics flavour), and a third analytics need would justify a service split. Two does not.

**Materialized view extension**

The existing pattern (FR7.2 — concurrent refresh to avoid read-locks) extends cleanly. New views:

- `mv_workspace_outcome_stats` — workspace × month × {bids tracked, submitted, won, win_rate, platform_attributed_count, avg_value, ...}
- `mv_workspace_content_reuse_stats` — workspace × content_block × usage_count × win_rate_when_used
- `mv_workspace_onboarding_milestones` — workspace × milestone × completed_at

Refresh cadence: hourly for milestones, daily for outcome stats.

**PDF generation**

Reuse WeasyPrint pipeline from Epic 7. The Outcome Brief is a structured monthly report — well within the existing pattern. **One trade-off:** Generating the PDF on-demand vs. monthly batch. Monthly batch (1st of each month, Celery Beat task in `notification` service) is simpler and cheaper. On-demand for ad-hoc renewal conversations can be added later.

**Configurability**

Per-workspace overrides on hourly rate and hours-saved-per-bid baseline — straightforward `client.workspace_outcome_config` table, defaults from `tenant_outcome_config`. Sensible defaults: €60K loaded cost, 19% search-time savings (per research §Step 2 source citations).

**API for QBR embedding (FR9.5)**

Pro+ tier-gated REST endpoint at `api.eusolicit.com/v1/workspaces/{id}/outcome` returning JSON. Aligned with existing public-API pattern (Sol Arch v5 §3.1 Enterprise public API).

**Onboarding milestone tracking (FR9.6)**

New `client.onboarding_milestones` table with workspace × milestone-type × completed-at columns. Domain events fire from existing flows (workspace created, content uploaded, first opportunity reviewed, first AI summary, CRM connected, first bid/no-bid). CSM alert on stalls > 7 days via `notification` service.

**Epic 19 sub-story arc (3 stories)**

| # | Story |
|---|---|
| **S19.00** | Outcome event capture (platform-attributed flag on opportunities, bid outcome capture UI, materialized views) |
| **S19.01** | Outcome Dashboard frontend widgets + workspace-config overrides |
| **S19.02** | Monthly Outcome Brief PDF generation + onboarding milestone tracker + CSM stall alerts |

---

### Change 7 — GTM functional requirements (NPS, reviews, onboarding)

**Verdict: 🟢 Feasible as stated. Use a third-party SDK for NPS to keep audit-trail clean.**

**NPS implementation**

Mary's recommendation — third-party SDK (Delighted, Wootric, or similar) — is correct for ISO 27001 evidence purposes. The vendor manages survey storage, response handling, and audit trail. We provide opt-in disclosure in the privacy notice and route the customer reference into our own analytics for opt-in routing to G2/Capterra.

**Trade-off:** Third-party adds a sub-processor to our list, and one more vendor to audit. Worth it for not building survey infrastructure.

**Onboarding milestone tracking** is already covered in Change 6 / FR9.6 / S19.02. No additional stories needed.

**Epic 20 sub-story arc (1 story — the rest is in Epic 19)**

| # | Story |
|---|---|
| **S20.00** | NPS SDK integration + opt-in routing to G2/Capterra deep links + privacy disclosure |

---

## 3. Cross-cutting architectural decisions

### CC-1: Tenant model migration — atomic switch

**Decision:** Yes, ship as **S14.00** with atomic data backfill. All existing companies become single-default-workspace tenants in one transaction. Pre-flight smoke test confirms backfill correctness; rollback plan is "drop the new column, restore the old foreign keys" — which is a 60-second operation if the migration is non-destructive.

**Recommendation:** Run the migration during a planned maintenance window (advance announce ≥48h). Use `pg_repack`-style logical replication pattern if the existing data volume requires zero-downtime. For the project's current scale (low data volume), a 15-minute maintenance window is acceptable.

### CC-2: Tier-gate matrix update

**Decision:** TierGate Depends() pattern from Epic 6 (project-context.md) extends cleanly. The matrix becomes:

```
TierGate(min_tier=Tier.PROFESSIONAL)  # gate by tier as before
    ↓ if pass
UsageGate(feature=Feature.AI_SUMMARY)  # gate by usage limit
    ↓ checks _USAGE_LUA, which now ALSO checks add_on_purchases for opportunity_id in scope
    ↓ if add-on active for opp → bypass tier cap
```

**No refactor required.** Pro+ adds one row to `tier_access_policies` and one enum value to `subscription.tier`. Existing TierGate code consumes both.

### CC-3: Service decomposition

**Decision:** **+1 net-new service: `integrations-api`.**

| Service | Status | Rationale |
|---|---|---|
| client-api | Existing — extends with workspace scope, FR9 outcome telemetry, FR2.4–2.7 metering bypass | Boring tech wins |
| admin-api | Existing — minimal changes | — |
| data-pipeline | Existing — no changes | — |
| ai-gateway | Existing — no changes | — |
| notification | Existing — extends with sub-processor change emails (FR10.4), onboarding stall alerts (FR9.6) | Reuses email pipeline |
| **integrations-api (NEW)** | New service | CRM + Slack/Teams; isolation, rate-limiting, sync workers |

**Decomposition stays at 6 services post-amendment.** Still well within the "5–10 sweet spot" the original decomposition analysis defines.

### CC-4: Frontend impact

**Decision:** Significant refactor in S14.03. New top-level workspace switcher, route changes (`/[locale]/workspace/[workspaceId]/...`), Zustand persist namespace migration. **Two ATDD-heavy sprints.** No avoidance possible — workspace UX is the consulting-firm value prop.

**One mitigation:** Ship S14.00, S14.01, S14.02 (backend) before S14.03 (frontend). The backend supports both old (no workspace) and new (workspace-scoped) request shapes during a transition window. Frontend cuts over in S14.03 with feature flag for staged rollout.

**Locale routing** stays per existing pattern (`apps/client/app/[locale]/...`); workspace becomes a sub-segment.

### CC-5: Backwards compatibility

**Decision (assumes no existing paying customers — flag for Deb):** Atomic migration in S14.00 + transition window in backend. If existing customers exist, we add a **forced-migration acceptance flow** at next login: "We've added client workspaces. Your existing data is now in your 'Default' workspace. [Got it]". This is a one-time UX event, not a long-lived shim.

**If Deb confirms existing paying customers exist:** S14.00 needs additional sub-tasks for customer communication, scheduled migration window, and customer-success-led migration assistance.

### CC-6: ISO 27001 audit dependency chain

**Decision:** Trust Center M2 (Epic 18 ships first), audit-prep starts M3 in parallel with Epics 14/15/19/16/17, audit kickoff M6, certification M12. Sequencer not blocker — feature epics do not wait on audit completion.

---

## 4. Refined data-model deltas

A consolidated view of new tables / columns required across the amendment:

```sql
-- Schema: client (Client API ownership)

-- New tables
CREATE TABLE client.client_workspaces (
    id              UUID PRIMARY KEY,
    company_id      UUID REFERENCES client.companies(id) NOT NULL,  -- tenant
    name            TEXT NOT NULL,
    archived_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (company_id, name) WHERE archived_at IS NULL
);

CREATE TABLE client.workspace_memberships (
    workspace_id    UUID REFERENCES client.client_workspaces(id),
    user_id         UUID REFERENCES client.users(id),
    role            TEXT NOT NULL,  -- admin | bid_manager | contributor | reviewer | read_only
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (workspace_id, user_id)
);

CREATE TABLE client.external_collaborators (
    id              UUID PRIMARY KEY,
    workspace_id    UUID REFERENCES client.client_workspaces(id) NOT NULL,
    proposal_id     UUID REFERENCES client.proposals(id) NOT NULL,
    email           TEXT NOT NULL,
    magic_link_jti  TEXT NOT NULL UNIQUE,
    expires_at      TIMESTAMPTZ NOT NULL,
    accepted_at     TIMESTAMPTZ,
    role            TEXT NOT NULL CHECK (role IN ('read_only', 'comment_only')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client.crm_connections (
    id              UUID PRIMARY KEY,
    workspace_id    UUID REFERENCES client.client_workspaces(id) NOT NULL,
    provider        TEXT NOT NULL CHECK (provider IN ('hubspot', 'salesforce', 'pipedrive')),
    encrypted_oauth TEXT NOT NULL,  -- Fernet-encrypted token blob
    status          TEXT NOT NULL DEFAULT 'active',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, provider)
);

CREATE TABLE client.workspace_outcome_config (
    workspace_id            UUID PRIMARY KEY REFERENCES client.client_workspaces(id),
    hourly_rate_eur         INTEGER,  -- nullable; falls back to tenant default
    hours_saved_per_bid     INTEGER,  -- nullable; falls back to tenant default
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client.onboarding_milestones (
    workspace_id    UUID REFERENCES client.client_workspaces(id),
    milestone       TEXT NOT NULL,  -- workspace_created | content_uploaded | first_opp | first_ai | crm_connected | first_bid_decision
    completed_at    TIMESTAMPTZ,
    PRIMARY KEY (workspace_id, milestone)
);

CREATE TABLE client.add_on_pricing_tiers (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,  -- standard | grant_module | enterprise_stack
    price_eur_cents INTEGER NOT NULL,
    stripe_price_id TEXT NOT NULL UNIQUE,
    feature_stack   JSONB NOT NULL,  -- which features this tier unlocks
    active          BOOLEAN NOT NULL DEFAULT true
);

-- New columns on existing tables
ALTER TABLE client.opportunities ADD COLUMN workspace_id UUID;  -- NULL = legacy/unmigrated
ALTER TABLE client.proposals ADD COLUMN workspace_id UUID;
ALTER TABLE client.documents ADD COLUMN workspace_id UUID;
ALTER TABLE client.content_blocks ADD COLUMN workspace_id UUID;
ALTER TABLE client.content_blocks ADD COLUMN tenant_scope BOOLEAN DEFAULT FALSE;  -- shared across workspaces
ALTER TABLE client.add_on_purchases ADD COLUMN add_on_pricing_tier_id UUID;
ALTER TABLE client.opportunities ADD COLUMN platform_attributed BOOLEAN DEFAULT FALSE;
ALTER TABLE client.bid_outcomes ADD COLUMN evaluator_score INTEGER, ADD COLUMN contract_value_eur INTEGER;

-- Schema: shared
ALTER TABLE shared.audit_log ADD COLUMN workspace_id UUID;  -- nullable

-- New materialized views
CREATE MATERIALIZED VIEW client.mv_workspace_outcome_stats AS ...;
CREATE MATERIALIZED VIEW client.mv_workspace_content_reuse_stats AS ...;

-- Schema: integrations (NEW — owned by integrations-api)
CREATE SCHEMA integrations;
CREATE TABLE integrations.sync_logs (
    id                  UUID PRIMARY KEY,
    crm_connection_id   UUID NOT NULL,
    direction           TEXT NOT NULL CHECK (direction IN ('inbound', 'outbound')),
    entity_type         TEXT NOT NULL,
    entity_id           TEXT NOT NULL,
    status              TEXT NOT NULL,
    error               TEXT,
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE integrations.conflict_log (
    id                  UUID PRIMARY KEY,
    crm_connection_id   UUID NOT NULL,
    entity_id           TEXT NOT NULL,
    eu_solicit_value    JSONB NOT NULL,
    crm_value           JSONB NOT NULL,
    resolution          TEXT NOT NULL,  -- lww_won_local | lww_won_remote
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Schema isolation note:** New `integrations` schema is owned by `integrations-api` service role. `integrations_api_role` has CRUD on `integrations` only and read-only on `client.crm_connections` (token vault stays in client schema for billing-scope reasons). Audit log entries written to `shared.audit_log` per existing pattern.

---

## 5. Updated service architecture diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                       │
└──┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┬───────┘
   │          │          │          │          │          │          │
 CloudFlare  Google    Microsoft  KraftData  HubSpot  Salesforce  Pipedrive
                                              + Slack + MS Teams (webhooks)
                                                       │
                                            ┌──────────▼──────────┐
                                            │ NEW: integrations-  │
                                            │      api service    │
                                            │  (FastAPI + Celery) │
                                            │  OAuth vault        │
                                            │  Sync engine        │
                                            │  Conflict log       │
                                            │  Circuit-breaker    │
                                            └──────────┬──────────┘
                                                       │
 ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──┴──────────┐  ┌─────────────┐
 │ SVC 1:      │  │ SVC 2:      │  │ SVC 3:      │  │ SVC 4:      │  │ SVC 5:      │
 │ CLIENT API  │  │ ADMIN API   │  │ DATA        │  │ AI          │  │ NOTIFICATION│
 │ + workspace │  │             │  │ PIPELINE    │  │ GATEWAY     │  │ + SP-change │
 │ + outcome   │  │             │  │             │  │             │  │   emails    │
 │   telemetry │  │             │  │             │  │             │  │ + onb stall │
 │ + per-bid   │  │             │  │             │  │             │  │   alerts    │
 │   metering  │  │             │  │             │  │             │  │             │
 └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘

 Database additions:
 ─ client schema:        +6 tables, +8 columns
 ─ shared.audit_log:     +1 column (workspace_id nullable)
 ─ integrations schema:  NEW — 2 tables

 Frontend additions:
 ─ /[locale]/workspace/[workspaceId]/...   ← all protected routes
 ─ Workspace switcher (top-level)
 ─ Trust Center: /trust (public, no auth)
 ─ NPS prompt SDK
```

---

## 6. Infra-impact estimate

**Engineering effort, calendar-quarter rough-order-of-magnitude:**

| Workstream | Effort | Calendar |
|---|---|---|
| Epic 14 — Multi-Client Workspace (5 stories, FE-heavy) | ~6–8 sprint-weeks | M0–M3 |
| Epic 15 — Per-Bid SKU + Pro+ Tier (3 stories) | ~3 sprint-weeks | M2–M3 (parallel with 14 tail) |
| Epic 18 — Trust Center (3 stories) | ~3 sprint-weeks | M0–M1 (parallel with 14 head) |
| Epic 19 — Outcome Telemetry (3 stories) | ~3 sprint-weeks | M3–M4 (after 14) |
| Epic 16 — Slack/Teams (1 story) | ~1 sprint-week | M3–M4 (parallel with 19) |
| Epic 17 — CRM (4 stories, 1 per provider) | ~10–12 sprint-weeks | M4–M7 |
| Epic 20 — NPS (1 story) | ~1 sprint-week | M5 |
| **Epic 21 — Platform Reliability for 99.9%** (6 platform-eng stories) | ~6–8 sprint-weeks | M0–M3 (parallel with feature epics) |
| **ISO 27001 audit-prep programme** (sprint-spanning) | ~8–10 sprint-weeks distributed | M2–M12 |

**Total feature engineering: ~28–35 sprint-weeks (≈ 2 quarters at 1 engineer; 1 quarter at 2 engineers).**

**New infrastructure costs (monthly):**

- Managed PostgreSQL Multi-AZ (vs. single-instance): +€100–200/mo at current scale
- Managed Redis HA (vs. single-instance): +€50–100/mo
- Third-party NPS SDK (Delighted/Wootric): €100–300/mo
- ISO 27001 audit cost (one-time + annual surveillance): €30–80K all-in (per Mary's amendment)
- New service (`integrations-api`) — minimal pod cost (~€20/mo)

**Total run-rate increase: ~€250–600/mo + ISO audit one-time.**

---

## 7. Refined epic sequencing

Mary's sequencing is mostly right. My refinements:

| Order | Epic | Stories | Dependencies | Calendar |
|---|---|---|---|---|
| **0** | **Epic 21 — Platform Reliability for 99.9%** (parallel platform workstream) | PE.01–PE.06 (6 stories) | k6 baseline carry-forward closes first | M0–M3 |
| **1** | **Epic 14 — Multi-Client Workspace** (foundational) | S14.00–S14.04 (5 stories) | Blocks 15, 17, 19, 20 | M0–M3 |
| **2** | **Epic 18 — Trust Center & Compliance Posture** | S18.00–S18.02 (3 stories) | Independent — parallel to 14 | M0–M1 |
| **3** | **ISO 27001 audit-prep programme** (sprint-spanning) | Cross-cutting evidence + runbooks + DR test | Epic 18 | M2–M12 |
| **4** | **Epic 15 — Per-Bid SKU + Pro+ Tier** | S15.00–S15.02 (3 stories) | Epic 14 (workspace billing scope) | M2–M3 |
| **5** | **Epic 16 — Slack/Teams Notifications** *(reordered from Mary's #6)* | S16.00 (1 story) | Epic 14, Epic 15 | M3–M4 |
| **6** | **Epic 19 — Outcome Telemetry** | S19.00–S19.02 (3 stories) | Epic 14 | M3–M4 |
| **7** | **Epic 17 — CRM Integrations** *(reordered from Mary's #5)* | S17.00–S17.03 (4 stories) | Epic 14, Epic 15 | M4–M7 |
| **8** | **Epic 20 — NPS, Reviews & Onboarding (residual)** | S20.00 (1 story; rest in Epic 19) | Epic 14, Epic 19 | M5 |

**Key changes vs. Mary's order:**
- **Added Epic 21** (Platform Reliability for 99.9%) as a parallel platform workstream with explicit stories
- **Swapped Slack/Teams ahead of CRM** — smaller scope, faster value to bid teams, larger addressable tier (Pro vs. Pro+)
- **Surfaced ISO 27001 as a separate sprint-spanning programme** rather than implicit in Epic 18

---

## 8. Risks not surfaced in the amendment

| # | Risk | Why surfaced | Mitigation |
|---|---|---|---|
| **1** | **KraftData EU residency / GDPR sub-processor evidence** | ISO 27001 audit will require auditable evidence that KraftData processes EU data in EU regions with a signed Data Processing Agreement. If KraftData hosts in a non-EU region or has unclear DPA, audit fails | Confirm KraftData EU residency + DPA before Epic 18 ships; add to sub-processor list |
| **2** | **Vector store cost scaling under per-workspace partitioning** | Per-workspace vector stores in KraftData = N × M storage cost. Per RB v5 §8, vector stores are per-storage-resource, billed by KraftData | Confirm KraftData billing model for many-small vs. few-large vector stores; consider tenant-scoped vector stores with workspace-tagged content as fallback |
| **3** | **External collaborator GDPR data flow** | When a consulting firm invites their client (an external organisation) as a collaborator, EU Solicit becomes a sub-processor of the consulting firm for that client's data. This requires DPA-of-a-DPA documentation | Update DPA template to clarify chain-of-processing; flag in legal review |
| **4** | **Stripe per-bid SKU + EU VAT MOSS reporting** | Stripe Tax handles VAT for one-time charges, but per-bid SKUs need separate product mapping for accurate VAT reporting | Confirm Stripe Tax product configuration; reuse Epic 8 patterns |
| **5** | **Multi-currency and BG market** | Pro+ at €199–399/user/month — but BG buyers may prefer BGN settlement. Stripe supports BGN but currency setup is non-trivial | Confirm Stripe currency setup early in Epic 15 |
| **6** | **ARR vs. one-time revenue tracking** | Per-bid SKU is one-time revenue, not ARR. Investor reporting and renewal-conversation modelling needs distinct tracking | Add reporting category to internal billing dashboards; not customer-visible |
| **7** | **Workspace deletion vs. GDPR Right to Erasure** | Audit log entries are immutable per project-context. When a workspace is deleted, audit entries persist. GDPR Art. 17 requires erasure of personal data — but audit logs are arguably "necessary for compliance with a legal obligation" (Art. 17.3.b) | Document the legal-basis exception; ensure audit entries reference workspace_id but not free-form PII |
| **8** | **Cross-workspace data leakage in shared content blocks** | Mary's amendment doesn't address this; my evaluation does (S14.05 / FR8.4 cross-workspace search). When a tenant-admin shares a content block across workspaces, the block can quote PII from one workspace into another | Tag content blocks as "tenant_scope=true" only after explicit user opt-in; PII detection at upload (deferred to Epic 19/20 or Epic 14.05 if scoped in) |
| **9** | **k6 baseline closure is on the critical path now** | Has been deferred 6 epics. Cannot publish 99.9% SLA without it. Cannot decompose Epic 21 without it | Make k6 closure the very first PE.01 story, before any other Epic 21 work |
| **10** | **CRM provider rate-limit surprises at scale** | HubSpot's 100 calls / 10 sec limit is fine for 50 customers; at 500 it gets tight. Salesforce 24-hour quota is harder to predict | Build per-provider Celery worker queues with backoff; plan for "rate-limit reached, paused for N minutes" UX in workspace integration view |

---

## 9. Open questions blocking implementation

Mary surfaced 6 open questions; my evaluation depends on three of them. Pulling them forward:

| # | Question | Why I'm blocked without it |
|---|---|---|
| **1** | **Are there existing paying customers from v1.0 PRD execution?** | Determines whether S14.00 needs additional customer-communication and migration-assistance sub-tasks. **Most blocking question.** |
| **2** | **What's the Pro+ price point?** (Mary's range: €199–399/user/month) | Need final number for Stripe Price ID configuration in S15.00 and tier-cache invalidation logic |
| **6** | **CRM provider order: HubSpot first, then Pipedrive, then Salesforce — confirmed?** | Determines Epic 17 sub-story sequence. Architecturally I'd hold this order regardless of market preference, due to complexity-driven sequencing |

Three other open questions (per-bid SKU pricing, Trust Center deadline, ISO 27001 budget) don't block architectural decomposition but **do block sprint-change-proposal generation** in `bmad-correct-course`. Resolve before that step.

---

## 10. Summary: where this lands

**The amendment is feasible. The architecture survives intact. The work is real but bounded.** Six PRD-level changes decompose into:

- **8 epics** (Epics 14–21, with 17 reordered ahead of 16)
- **~22 stories** (S14.00–S14.04, S15.00–S15.02, S18.00–S18.02, S16.00, S19.00–S19.02, S17.00–S17.03, S20.00, PE.01–PE.06)
- **+1 net-new service** (`integrations-api`)
- **+6 new tables** in `client` schema, +1 schema (`integrations`), +1 nullable column on `shared.audit_log`
- **2 quarters of platform engineering** (1 quarter at 2 engineers); +€250–600/mo run-rate; +€30–80K one-time ISO 27001 audit
- **No redesign of v5 architecture.** Patterns from Epics 1–13 extend cleanly.

The single largest architectural risk is **multi-client workspace migration** (S14.00 + frontend refactor in S14.03), and the largest programme risk is **99.9% SLA backed by infrastructure not yet built** (Epic 21). Both are surfaced in this evaluation with concrete sub-stories.

**Next step (recommended):** `bmad-correct-course` to produce the formal sprint-change-proposal, then `bmad-create-epics-and-stories` to decompose Epics 14–21 into the sprint plan.

---

## Document status

| Item | Status |
|---|---|
| Source research | Complete (`research/market-eu-govwin-tender-intelligence-research-2026-04-25.md`) |
| PRD amendment brief | Complete (`prd-amendment-2026-04-25.md`) |
| Canonical PRD update (v1.1) | Complete |
| **Architecture evaluation (this document)** | **Approved v1.0 — Deb signed off on all six open questions on 2026-04-25 (see §11)** |
| Sprint-change-proposal | In-flight (`bmad-correct-course`) |
| Epic decomposition | Pending (`bmad-create-epics-and-stories`) |

**Author:** Winston (System Architect)
**Date:** 2026-04-25
**Review owner:** Deb — signed off on all six open questions on 2026-04-25 (see §11). Hand-off to `bmad-correct-course` for sprint-change-proposal.

---

## 11. Locked decisions (signed off 2026-04-25 by Deb)

Deb agreed to the architecture-evaluation defaults and Mary's recommendations on all six open questions. Decisions locked for sprint-change-proposal generation and epic decomposition:

| # | Question | Locked decision | Implication |
|---|---|---|---|
| **1** | **Existing paying customers from v1.0?** | **None.** | S14.00 ships as atomic migration only — no customer-communication sub-tasks, no scheduled-migration window, no CSM-led migration assistance. Maintenance window for the migration is acceptable (15 minutes, advance announce ≥48h not customer-facing required since no customers exist) |
| **2** | **Pro+ tier price point** | **€199/user/month** (lower end of €199–399 range) | Stripe Price ID configured at €199. Rationale: lower entry price captures more consulting firms during BG/CEE beachhead; raise based on Phase-1 customer feedback (M9–M12). Locked S15.00 acceptance criterion. |
| **3** | **Per-bid SKU pricing tiers** | **€99 standard / €199 with grant module / €299 full Enterprise feature stack** | Three Stripe Price IDs in `infra/stripe-config.yaml`. `add_on_pricing_tiers` table seeded at deploy time. Locked S15.01 acceptance criterion. |
| **4** | **Trust Center launch deadline** | **Month 2** of platform launch | Epic 18 sequenced for M0–M1 calendar, page goes live by M2. Hard deadline — gates initial mid-large enterprise sales conversations. |
| **5** | **ISO 27001 audit budget** | **€50K** planning assumption (midpoint of €30–80K range) | Auditor engagement scoped accordingly: Stage 1 + Stage 2 + remediation cycle within €50K. Surveillance audit (Year 2) budget separate. Engineering effort scoped at ~8–10 sprint-weeks across M2–M12. |
| **6** | **CRM provider order** | **HubSpot → Pipedrive → Salesforce** | Confirmed both market-driven (research §Step 4 — BG/CEE/SME density) and complexity-driven (Salesforce 6-week add). Locked Epic 17 sub-story sequence S17.00 → S17.01 (HubSpot) → S17.02 (Pipedrive) → S17.03 (Salesforce). |

### Resolved cross-cutting decision (CC-5)

With "no existing paying customers" locked, **CC-5 (backwards compatibility) simplifies materially**: S14.00 ships as a clean atomic migration with no transition-window UX or forced-migration acceptance flow. The backend supports only the new (workspace-scoped) request shape from cutover; no dual-mode shim required. **This removes one risk class from the largest epic.**

### What this unlocks

- ✅ `bmad-correct-course` can now generate the formal sprint-change-proposal
- ✅ `bmad-create-epics-and-stories` can decompose Epics 14–21 with all pricing/sequencing/scope locked
- ✅ Stripe configuration work (3 product IDs for per-bid + 1 for Pro+) can be scheduled into S15.00–S15.01
- ✅ Epic 18 (Trust Center) can begin immediately in parallel with Epic 14
- ✅ Auditor RFP can be issued against the €50K budget envelope in M3

