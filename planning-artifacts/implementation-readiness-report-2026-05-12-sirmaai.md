---
stepsCompleted: ["step-01-document-discovery", "step-02-prd-analysis", "step-03-epic-coverage-validation", "step-04-ux-alignment", "step-05-epic-quality-review", "step-06-final-assessment"]
documentsAssessed:
  - "PRD.md (live) + prd-amendment-2026-05-12-sirmaai.md (delta, treat as merged)"
  - "architecture.md (live) + architecture-amendment-2026-05-12-sirmaai.md (delta with ADR-018/019/020 + ADR-004 addendum, treat as merged)"
  - "epics/E04-ai-gateway-service.md (2026-05-12 amendment appended)"
  - "epics/E05-data-pipeline-ingestion.md (2026-05-12 amendment appended)"
  - "epics/E11-grants-compliance.md (2026-05-12 amendment appended)"
  - "epics/E17-crm-integrations.md (2026-05-12 amendment appended)"
  - "epics/E24-sirmaai-tenant-provisioning.md (new)"
  - "epics/E25-knowledge-base-lifecycle.md (new)"
  - "epics/E26-agent-driven-ingestion.md (new)"
  - "epics/E27-crm-via-mcp.md (new — narrative; delivery via E17 amendment)"
  - "epics/E28-webhook-reconciliation.md (new)"
  - "sprint-change-proposal-2026-05-12-sirmaai.md (approved 2026-05-12)"
verdict: "PASS (after gap-closing pass 2026-05-12)"
priorVerdict: "CONCERNS (initial assessment; 10 gaps identified)"
gapClosurePass: "2026-05-12"
---

# Implementation Readiness Assessment Report

**Date:** 2026-05-12
**Project:** EU Solicit
**Assessment scope:** Post-SirmaAI integration pivot — pre-launch re-platform of the AI substrate
**Assessor:** 📋 John (PM, in expert-PM readiness role)
**Initial verdict (first pass):** CONCERNS — 10 specific gaps identified
**Final verdict (after gap-closing pass on 2026-05-12):** **✅ PASS — cleared for orchestrator sprint replan**

---

## Executive Summary

The post-pivot artefact set (PRD amendment + architecture amendment + 4 modified epics + 5 new epics) is **structurally coherent** and **traceable**. The locked decisions (Topology A, scope swap on CRM, 5 new epic injection) flow consistently from sprint change proposal → PRD amendment → architecture amendment → epic delta. All FR/NFR additions have epic owners. ADRs are referenced consistently. The dependency chain (E04 amendment → E24 → E25 → E26 → E27/E28) is sound with no circular references and no forward-only dependencies.

**However, the validation surfaces 10 specific gaps** ranging from missing migration ownership to a backfill story for pre-pivot existing companies. These are **all closable with story-level adjustments** to existing epic files — no architectural rework required. Address gaps 1, 2, 4, and 6 (the four MAJOR concerns) before orchestrator dispatch; the six MINOR concerns can land during sprint replan as story-level refinements.

**Recommended verdict transition to PASS:** close the 4 major gaps via targeted story injections (~1 sprint of PM/Dev work), then re-run readiness. Estimated 2-3 days of additional planning work before orchestrator pickup.

---

## Document Inventory

| Document | Path | Notes |
|---|---|---|
| PRD (live) | `planning-artifacts/PRD.md` | FR-1..FR-44, NFR-1..NFR-23 — pre-pivot baseline |
| PRD amendment | `planning-artifacts/prd-amendment-2026-05-12-sirmaai.md` | FR-45..FR-55, NFR-24..NFR-26 — post-pivot additions |
| Architecture (live) | `planning-artifacts/architecture.md` | v2.0; ADR-001..ADR-017 |
| Architecture amendment | `planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` | ADR-018, ADR-019, ADR-020, ADR-004 addendum; §1.1, §1.2, §2.1, §2.2, §3.4, §4.1, §4.4, §5.1, §5.3, §8.1, §11.3 deltas |
| Epic E04 (amended) | `planning-artifacts/epics/E04-ai-gateway-service.md` | Original `done`; 2026-05-12 Amendment appended with S04.20–S04.27 |
| Epic E05 (amended) | `planning-artifacts/epics/E05-data-pipeline-ingestion.md` | Original `done`; 2026-05-12 Amendment appended with S05.20–S05.25 |
| Epic E11 (amended) | `planning-artifacts/epics/E11-grants-compliance.md` | Original `done`; 2026-05-12 Amendment appended with S11.20–S11.23 |
| Epic E17 (amended) | `planning-artifacts/epics/E17-crm-integrations.md` | Original `done`; 2026-05-12 Amendment appended with S17.30–S17.36 |
| Epic E24 (new) | `planning-artifacts/epics/E24-sirmaai-tenant-provisioning.md` | 7 stories, 21 pts |
| Epic E25 (new) | `planning-artifacts/epics/E25-knowledge-base-lifecycle.md` | 9 stories, 34 pts |
| Epic E26 (new) | `planning-artifacts/epics/E26-agent-driven-ingestion.md` | 10 stories, 34 pts |
| Epic E27 (new) | `planning-artifacts/epics/E27-crm-via-mcp.md` | Narrative; delivery via E17 amendment; ~34 pts (shared) |
| Epic E28 (new) | `planning-artifacts/epics/E28-webhook-reconciliation.md` | 8 stories, 21 pts |
| Sprint change proposal | `planning-artifacts/sprint-change-proposal-2026-05-12-sirmaai.md` | Approved 2026-05-12 |

**No duplicates.** No sharded versions. No missing required documents. PRD + Architecture + Epics all present and amended consistently.

---

## PRD Analysis — New Requirements from Amendment

Per scope constraint (the existing 23-epic body is `done` and unaffected by the pivot at the FR-1..FR-44 / NFR-1..NFR-23 layer), this validation focuses on the **new** requirements introduced by the PRD amendment.

### New Functional Requirements (from PRD amendment)

| # | Summary |
|---|---|
| **FR-45** | Auto-create SirmaAI Project on EU Solicit company create |
| **FR-46** | Soft-delete SirmaAI Project on EU Solicit company archive |
| **FR-15 (amended)** | Ingestion via N8N + SirmaAI crawler agents (replaces Celery crawl) |
| **FR-47** | Opportunity Qualification Analysis (SirmaAI agent, structured output) |
| **FR-48** | Opportunity Quantification Analysis (SirmaAI agent, async-run) |
| **FR-49** | KB artefact upload (tier-gated quotas) |
| **FR-50** | KB semantic search |
| **FR-51** | Agent grounding in KB with citation surface |
| **FR-52** | KB lifecycle (replace, archive, re-index on profile update) |
| **FR-53** | CRM via SirmaAI MCP servers (Dynamics 365 + HubSpot v1) |
| **FR-54** | Standard Webhooks receiver (HMAC, idempotency, DLQ) |
| **FR-55** | 5-min run-state reconciler |
| **FR-36 (amended)** | ESPD via SirmaAI agent; EU Solicit retains XML/PDF render |

### New Non-Functional Requirements (from PRD amendment)

| # | Summary |
|---|---|
| **NFR-2 (amended)** | TTFB <500ms SLO scoped to synchronous-stream paths only; async-run governed by SirmaAI job-engine latency |
| **NFR-6 (amended)** | Fernet-encryption-at-app-layer scope extended to per-tenant SirmaAI Project API keys, CRM OAuth tokens, webhook HMAC secrets, SirmaAI system key |
| **NFR-24** | Per-Project SirmaAI API key rotation on 90-day cadence with double-validation overlap |
| **NFR-25** | Tier upgrades/downgrades mirrored to SirmaAI Org rate-limit policy within 60s |
| **NFR-26** | SirmaAI as critical external dependency; two-layer resilience + degraded-mode banner + reconciler-as-authoritative-truth |

---

## Epic Coverage Validation — FR/NFR Traceability

| FR / NFR | Owning Epic | Owning Story (or stories) | Status |
|---|---|---|---|
| FR-45 (provisioning) | E24 | S24.01 (schema), S24.02 (creation), S24.03 (template seed), S24.04 (MCP stubs) | ✅ Covered |
| FR-46 (archival) | E24 | S24.06 (archive flow), S24.07 (R2E cross-substrate proof) | ✅ Covered |
| FR-15 amended (N8N ingestion) | E05 amendment + E26 | E05 S05.20 (workflow templates), S05.21 (webhook → opportunity writer), E26 S26.03 (analysis workflow) | ✅ Covered |
| FR-47 (qualification) | E26 | S26.01 (qualifier agent), S26.05 (trigger endpoint), S26.07 (webhook→DB) | ✅ Covered |
| FR-48 (quantification) | E26 | S26.02 (quantifier agent), S26.06 (auto-cascade) | ✅ Covered |
| FR-49 (KB upload) | E25 | S25.02 (upload proxy) | ✅ Covered |
| FR-50 (KB semantic search) | E25 | S25.04 (search proxy) | ✅ Covered |
| FR-51 (agent grounding + citations) | E25 + E26 | E25 S25.09 (citation surface), E26 S26.09 (KB-grounded regression) | ✅ Covered |
| FR-52 (KB lifecycle) | E25 | S25.05 (replace/archive/restore), S25.06 (profile-update re-index) | ✅ Covered |
| FR-53 (CRM via MCP) | E17 amendment / E27 | S17.30 Dynamics, S17.31 HubSpot, S17.32 OAuth+token push, S17.33 rotation, S17.34 conflict log, S17.35 tier gate, S17.36 archive cleanup | ✅ Covered |
| FR-54 (Standard Webhooks receiver) | E04 amendment + E28 | E04 S04.25 (receiver), E28 S28.01–S28.03 (subscription mgmt, rotation, hardening) | ✅ Covered |
| FR-55 (reconciler) | E04 amendment + E28 | E04 S04.26 (basic reconciler), E28 S28.05 (hardening + observability) | ✅ Covered |
| FR-36 amended (ESPD via SirmaAI) | E11 amendment | S11.20 (template seed), S11.22 (endpoint internal call swap) | ✅ Covered |
| NFR-2 amended (TTFB scope) | E04 amendment | S04.20–S04.24 (gateway refactor preserves SSE per ADR-005) | ✅ Covered |
| NFR-6 amended (Fernet scope) | E04 amendment + E17/E27 | E04 S04.22 (per-Project key vault), E17 S17.32 (CRM OAuth tokens), E28 S28.02 (HMAC secret rotation) | ✅ Covered |
| NFR-24 (key rotation 90-day) | E04 amendment | S04.22 (vault + rotation) | ✅ Covered |
| **NFR-25 (tier→rate-limit sync)** | **— UNOWNED —** | **No story in any epic** | **❌ GAP — see Concern #1** |
| NFR-26 (SirmaAI as external dep) | E04 + E28 | E04 S04.27 (degraded banner), E28 S28.05 (reconciler), S28.07 (banner) | ✅ Covered |

**Coverage statistics:**
- Total new FRs/NFRs: 18 (13 FR + 5 NFR)
- Covered: 17 / 18 (94.4%)
- Uncovered: **1 (NFR-25)** — see Concern #1

---

## Schema Table Ownership Validation

The architecture amendment §4.1 introduces 5 new tables plus 2 more authored in individual epics. Each needs an owning Alembic migration story.

| Table | Schema | DDL location | Migration story | Status |
|---|---|---|---|---|
| `client.sirmaai_projects` | client | arch amendment §4.1 | E04 S04.21 ("Alembic migration for `client.sirmaai_projects`") | ✅ Covered |
| `client.sirmaai_kb_files` | client | arch amendment §4.1 | E25 S25.01 ("Alembic migration for `client.sirmaai_kb_files`") | ✅ Covered |
| **`client.sirmaai_mcp_servers`** | client | arch amendment §4.1 | **No explicit migration story** (referenced as existing by E24 S24.04 + E17 S17.30) | **❌ GAP — see Concern #2** |
| **`gateway.webhook_subscriptions`** | gateway | arch amendment §4.1 | **No explicit migration story** (referenced as existing by E04 S04.25, E28 S28.02) | **❌ GAP — see Concern #2** |
| **`gateway.workflow_runs`** | gateway | arch amendment §4.1 | **No explicit migration story** (referenced as existing by E04 S04.24, E28 S28.05) | **❌ GAP — see Concern #2** |
| `client.opportunity_analyses` | client | E26 epic schema section | E26 S26.04 ("Alembic migration + SQLAlchemy model") | ✅ Covered |
| **`gateway.webhook_dlq`** | gateway | E28 epic schema section | **Implicit in E28 S28.06 but not explicit** | **❌ GAP — see Concern #2** |

---

## Cross-Cutting Invariant Validation

| Invariant | Status | Evidence |
|---|---|---|
| **ADR-018 (Topology A) referenced consistently** | ✅ Sound | PRD amendment §SaaS B2B Multi-Tenancy; arch §1.1, §3.4, §4.4, §8.1; epics E04 amendment, E24, E27 |
| **ADR-019 (KB ownership split) referenced consistently** | ✅ Sound | PRD amendment §FR-49..52, §Data Residency; arch §3.4, §4.4 ("KB artefact integrity"); epic E25, E24 S24.07 (R2E) |
| **ADR-020 (CRM via MCP) referenced consistently** | ✅ Sound | PRD amendment FR-53, §Integrations; arch §3.4, §5.1; epics E17 amendment, E27 |
| **E04 amendment ↔ E24 dependency wired correctly** | ✅ Sound | E24 explicitly declares "Dependencies: E04 amendment"; E24 stories reference S04.21 (schema), S04.22 (vault) |
| **E25 ↔ E26 dependency (grounding ↔ KB lifecycle)** | ✅ Sound | E26 declares dependency on E25; E26 S26.09 explicitly tests KB-grounded responses |
| **E27 ↔ E17 amendment — no scope drift** | ✅ Sound | E27 explicitly declares E17 amendment as the orchestrator-dispatch source; story IDs deliberately overlap (S27.0x = S17.3x); both name the same 5 MCP tools, same 7 stories, same Dynamics+HubSpot v1 + Pipedrive/SF deferral |
| **Cross-substrate R2E two-ACK pattern (ADR-019 ↔ E24 S24.07)** | ✅ Sound | ADR-019 specifies; arch §4.4 adds as data pattern; E24 S24.07 implements with `erasure_step` audit entries; nightly sweep catches stale erasures |
| **SirmaAI EU residency flagged as launch-blocking** | ✅ Sound | Arch §11.3 risk #1 elevated to "Launch-blocking"; PRD amendment §Data Residency; sprint change proposal Section 1.2 + Section 5; Winston's handoff brief; this report Concern #6 reminder |
| **N8N staged-rollout discipline lifted into story ACs** | ✅ Sound | ADR-018 *Consequences*; E05 S05.24 ("staged-rollout enforcement"); E26 S26.03 ("staged rollout: canary → 10% → 100%") |
| **Reconciler-as-authoritative + idempotency-guard discipline** | ✅ Sound | Arch §4.4 ("`UPDATE ... WHERE status IN ('pending','running')` guard"); E28 S28.04 explicitly documents the rule for all consumers |

---

## Epic Quality Review

### User Value vs Technical Milestone

| Epic | User value? | Notes |
|---|---|---|
| E24 Tenant Provisioning | ✅ Yes | Without this, no tenant can use AI features; user value = "platform works for me" |
| E25 KB Lifecycle | ✅ Yes | Direct user surface: upload artefacts, search, grounded agent outputs with citations |
| E26 Agent-Driven Ingestion | ✅ Yes | Qualification + quantification surface in opportunity detail; user-visible scored opportunities |
| E27 CRM via MCP | ✅ Yes | Bi-directional CRM connectivity; user benefit is enriched leads + auto-deal-sync |
| E28 Webhook & Reconciliation | ⚠️ Borderline (acceptable) | Operational hardening + degraded-mode banner (user-visible) + DLQ admin (operator-visible). Similar to authentication-system-borderline class; accepted because it enables every other epic's reliability |

### Independence and Forward-Dependency Check

| Epic | Independence | Forward-dep? |
|---|---|---|
| E04 amendment | Foundational — depends on no other amendment | None |
| E05 amendment | Depends on E04, E24, E25 (backward) | None |
| E11 amendment | Depends on E24, E25 (backward) | None |
| E17 amendment | Depends on E04, E24, E25 (backward) | None |
| E24 | Depends on E04 amendment (backward) | None |
| E25 | Depends on E24 (backward) | None |
| E26 | Depends on E04, E05 amend, E24, E25, E28 (all backward) | None |
| E27 | Depends on E04, E24, E25 (backward) | None |
| E28 | Depends on E04 amendment (parallel-trackable) | None |

**No circular dependencies. No forward-only dependencies.** Chain is sound.

### Story Sizing & Acceptance Criteria Spot-Check

Sampled 12 stories across new epics for AC quality (specific, testable, complete):

| Story | AC quality |
|---|---|
| E24 S24.02 (Project creation) | ✅ Specific (30s p95, idempotent re-invocation, cross-tenant negative) |
| E24 S24.05 (reconciliation) | ✅ Specific (daily 03:00 UTC, Prometheus gauge name, orphan list endpoint) |
| E24 S24.07 (cross-substrate R2E) | ✅ Specific (two-ACK audit pattern, 48h stale sweep) |
| E25 S25.02 (upload) | ✅ Specific (25MB PDF p95, incremental SHA-256, 402 with code on over-quota, 415 with allowed-types on wrong type) |
| E25 S25.07 (quota + dashboard) | ✅ Specific (Pro 5GB cap, WCAG 2.1 AA, Playwright E2E sequence) |
| E25 S25.08 (nightly hash reconcile) | ✅ Specific (10K files / 100 tenants in 10 min) |
| E26 S26.01 (qualifier agent) | ✅ Specific (≥85% eval-run alignment with human baseline) |
| E26 S26.06 (auto-cascade) | ✅ Specific (60s steady state, 100 concurrent throttle, pursue→quantification cascade) |
| E26 S26.10 (legacy equivalence) | ✅ Specific (50 opportunities, distinguish improvement vs regression, human sign-off gate) |
| E27 / S17.32 (OAuth + token push) | ✅ Specific (state nonce validation, Fernet encryption, secret-injection to SirmaAI) |
| E28 S28.03 (receiver hardening) | ✅ Specific (signature mismatch 401, duplicate 200, malformed → DLQ + 202) |
| E28 S28.05 (reconciler hardening) | ✅ Specific (5-min cadence at 10K rows, 429 backoff, alert threshold sustained 30 min) |

No vague-AC violations. No "user can do X" hand-waves spotted in the sample.

---

## Concerns and Gaps

### 🔴 MAJOR Concerns (4)

#### Concern #1 — NFR-25 (tier→rate-limit sync) has no owning story

**Gap:** PRD amendment NFR-25 mandates that tier upgrades/downgrades mirror to SirmaAI Org rate-limit policy via `PATCH /api/admin/organizations/{orgId}/rate-limit` within 60 seconds. **No epic or story owns this implementation.**

**Impact:** Functional drift between EU Solicit tier metering and SirmaAI's rate-limit enforcement. Pro+ tenant upgrades will not lift their SirmaAI rate ceiling; downgrades will not lower it.

**Remediation:**
- Inject **S04.28 Tier-to-SirmaAI-rate-limit sync** into E04 amendment.
- Subscribes to existing `subscription.changed` Redis Streams event (Epic 8 pattern).
- On event: lookup tier → rate-limit mapping (config-driven, default Free=100/day, Starter=1k/day, Professional=10k/day, Pro+ =20k/day, Enterprise=100k/day — subject to commercial calibration).
- `PATCH /api/admin/organizations/{orgId}/rate-limit` within 60s SLO.
- Idempotent retry on transient SirmaAI failures.
- Estimated 3 pts.

---

#### Concern #2 — Three schema migrations have no explicit owning story

**Gap:** Architecture amendment §4.1 defines `client.sirmaai_mcp_servers`, `gateway.webhook_subscriptions`, `gateway.workflow_runs`. Multiple stories *reference* these tables as existing (E04 S04.24/S04.25, E24 S24.04, E17 S17.30, E28 S28.02) but **no story explicitly authors the Alembic migration**. Similarly `gateway.webhook_dlq` (defined in E28 epic body) lacks an explicit migration story.

**Impact:** When Amelia picks up the first story, the schema isn't there. Risk of stories blocking each other or migrations being authored ad-hoc with inconsistent revision numbering.

**Remediation:**
- Expand **E04 S04.21** to include the full `gateway.*` schema migrations: `gateway.webhook_subscriptions`, `gateway.workflow_runs` (in addition to `client.sirmaai_projects`).
- Expand **E24 S24.01** to include `client.sirmaai_mcp_servers` migration alongside `client.sirmaai_projects`.
- Add explicit migration line item to **E28 S28.06** for `gateway.webhook_dlq`.
- No new stories needed; story descriptions just expand. ~1pt of additional scope per affected story.

---

#### Concern #4 — `sirmaai-gateway` public ingress contradicts original E04 ACs

**Gap:** Original E04 AC reads: *"Service runs as ClusterIP only — no Ingress, no public exposure."* Post-pivot, the webhook receiver MUST be publicly reachable for SirmaAI to deliver Standard Webhooks. Per memory, **on-prem launch ADR-010** means manual nginx configuration on www1 (per memory `project_deploy_nginx_manual.md`).

**Impact:** Either (a) the AC contradiction means deploy will fail review, or (b) the receiver will silently be unreachable and webhooks never arrive — reconciler then carries the full load (works, but defeats latency optimization).

**Remediation:**
- Inject **S04.29 Public ingress for webhook receiver** into E04 amendment:
  - nginx vhost on www1: `https://api.eusolicit.com/webhooks/sirmaai` → cluster-internal `sirmaai-gateway:8004/webhooks/sirmaai`.
  - Requires manual sudo cp on www1 per `project_deploy_nginx_manual.md`; runbook entry in `eusolicit-docs/runbooks/`.
  - TLS via existing certbot on www1; ACME challenge stays open in nginx config (existing pattern).
  - Update original E04 AC #15 to reflect that **the webhook receiver path is publicly exposed; all other gateway paths remain ClusterIP-only**.
- Estimated 3 pts.

---

#### Concern #6 — No backfill story for existing pre-pivot companies

**Gap:** E24 explicitly puts pre-pivot company migration "out of scope" and points at E22 onprem-launch close-out. But **E22 is `done` per sprint-status**. No epic or story owns the one-off backfill: when E24 ships, all currently-existing companies (and any created after E22 closed but before E24 ships) need SirmaAI Projects auto-created.

**Impact:** First customer-pilot tenant will hit "no SirmaAI Project — AI features unavailable" on day one of post-pivot launch.

**Remediation:**
- Inject **S24.08 Backfill SirmaAI Projects for existing companies** into E24:
  - One-off Celery task: enumerate all non-archived `client.companies` without `client.sirmaai_projects` row; invoke S24.02 provisioning flow for each.
  - Throttle: 1 concurrent provisioning per minute to avoid SirmaAI rate-limit storm.
  - Idempotent: re-run safe.
  - Backfill audit row written to `shared.audit_log` per provisioned tenant.
  - Operator-runbook entry covering: pre-flight check (count of companies), expected duration (companies × 60s ÷ throttle), monitoring during backfill, failure-recovery procedure.
- Estimated 3 pts.

---

### 🟠 MINOR Concerns (6)

#### Concern #3 — `eusolicit-kraftdata` → `eusolicit-sirmaai` package rename lacks an explicit story

**Gap:** Architecture amendment §2.2 calls out the package rename; E04 S04.27 mentions it inside an AC bullet (*"client packages...re-generated"*) but no story explicitly does the rename + consumer-service migration across `client-api`, `admin-api`, `data-pipeline`. Each consumer service has its own pyproject.toml dependency on `eusolicit-kraftdata`.

**Impact:** Probable build failures during the E04 amendment rollout if not coordinated.

**Remediation:** Expand E04 S04.27 (currently the "Tenant degraded-mode banner" story) — split into S04.27a (degraded banner) and S04.27b (package rename + consumer updates). Alternatively, inject a separate S04.30. ~3pt scope addition.

---

#### Concern #5 — `tenant.provisioned` event missing from Event Catalog §5.3

**Gap:** E24 publishes a `tenant.provisioned` Redis Streams event. Architecture amendment §5.3 catalogues new *inbound* SirmaAI webhook events but does not add the new *internal* `tenant.provisioned` event to the catalog.

**Impact:** Discovery friction — downstream consumers (notification welcome email, client-api UI badge update) may not be aware of the event without grepping epic ACs.

**Remediation:** Append to architecture amendment §5.3:
- `tenant.provisioned` → `notification` (welcome email enriched with KB-upload CTA), `client-api` (UI badge update for provisioning completion)

---

#### Concern #7 — N8N workflow template source-of-truth location not specified

**Gap:** ADR-018 *Consequences* requires workflow templates to be treated as "production code: PR review + rollback plan." E05 S05.20 and E26 S26.03 say "authored" but don't specify the **storage location** of the templates. Are they in EU Solicit's Git repo (e.g. `infra/n8n-templates/`)? Are they exported from SirmaAI's N8N UI and committed? Are they imported via SirmaAI API at deploy time?

**Impact:** Without a canonical source, "PR review + rollback" cannot be enforced. Templates risk drifting between staging and production N8N instances.

**Remediation:** Architectural call needed. Two options: (a) commit JSON export of each template to `infra/n8n-templates/` and apply via CI on deploy; (b) treat SirmaAI as the source-of-truth and add deploy-script that diffs SirmaAI templates against committed JSON and refuses to deploy on drift. Recommend **(a)** for simplicity. Add as **NEW S05.26** or amend S05.20 description to include the deploy-mechanism.

---

#### Concern #8 — Local-dev SirmaAI story unowned

**Gap:** ADR-019 *Consequences* names two options (point dev at staging vs build a `sirmaai-mock` thin FastAPI). Neither commits in an epic story.

**Impact:** Developer productivity. First dev who tries `make up` for the first time will discover the gap.

**Remediation:** Inject **S04.31 Local-dev SirmaAI stub** into E04 amendment (or add as a sub-task to S04.20). Recommend Option 1 (point at staging) for v1; defer mock to when isolation pain warrants. ~1pt. Document in `eusolicit-app/CLAUDE.md` or a runbook.

---

#### Concern #9 — `shared.audit_log.details JSONB` accommodation of new fields untested

**Gap:** ADR-019 + E24 S24.07 use new `erasure_step` semantics in audit_log. Likely stuffs into existing `details JSONB` column — but no story explicitly verifies the existing schema accommodates the new shape (e.g. indexing by `details->>'erasure_step'` for the nightly sweep).

**Impact:** Sweep job from S24.07 might do a full table scan or miss entries depending on JSONB indexing choices.

**Remediation:** Add explicit verification step to E24 S24.07 description: "Verify `shared.audit_log.details->>'erasure_step'` queries are performant; add GIN index on `details->>'erasure_step'` if EXPLAIN ANALYZE shows seq scan." No new story needed; AC tightening.

---

#### Concern #10 — Idle Free-tier tenant cost unmodeled

**Gap:** Per the sprint change proposal §"Three things flagged for your call", idle-tenant SirmaAI cost is unconfirmed. Topology A's free-tier acquisition economics depend on it. No story or PRD requirement specifies "Free-tier tenants get inactive Project (no AI)" vs "Free-tier tenants get active Project with capped quota" — this affects FR-45's behaviour for non-paying tenants.

**Impact:** Either (a) Free tier becomes a paid feature in disguise (if Projects always cost money to keep alive), or (b) Free tier provisioning succeeds but no AI capabilities work for free tenants (potentially fine but should be intentional).

**Remediation:** Product call required before E24 implementation. Captured here as the placeholder; raise with Deb in product alignment session before E24 sprint kickoff. May require a small FR addition.

---

## UX Alignment

This pivot does not introduce new UX surfaces beyond:
- Opportunity detail page additions for qualification + quantification panels (E26 S26.08 — covers WCAG 2.1 AA, Playwright E2E)
- KB management dashboard per workspace (E25 S25.07 — covers WCAG 2.1 AA, drag-and-drop keyboard fallback)
- CRM connection widget per workspace (E27 S17.35 — referenced)
- Degraded-mode banner (E28 S28.07 — WCAG 2.1 AA `role="status"` + `aria-live="polite"`)

The existing `EU_Solicit_UX_Supplement_v1.md` (in `eusolicit-docs/`) does not need amendment — new surfaces follow established design system patterns (shadcn/ui, TanStack Query + `<QueryGuard>`, TanStack-Query-via-SSE-invalidation per Epic 6).

**No UX-design gap surfaced.** ✅

---

## Summary and Recommendations

### Overall Readiness Status

**CONCERNS — Hold orchestrator dispatch until 4 MAJOR concerns are closed.**

### Critical Issues Requiring Immediate Action

1. **Concern #1 — NFR-25 unowned.** Inject S04.28 tier-to-rate-limit sync story into E04 amendment (~3 pts).
2. **Concern #2 — 3 schema migrations + 1 DLQ migration unowned.** Expand E04 S04.21 (gateway.*), E24 S24.01 (client.sirmaai_mcp_servers), E28 S28.06 (gateway.webhook_dlq). ~1pt scope addition each, no new stories.
3. **Concern #4 — Public ingress for webhook receiver contradicts original E04 AC.** Inject S04.29 public ingress story (~3 pts). Update original E04 AC #15 wording.
4. **Concern #6 — No backfill story for pre-pivot companies.** Inject S24.08 backfill story into E24 (~3 pts).

### Recommended Next Steps

1. **Triage product decisions (Deb):** Concern #10 (Free-tier Project policy) needs a product call. Concerns #1 and #6 may also benefit from a Deb sign-off on the proposed remediation (tier rate-limit mapping numbers; backfill throttle).
2. **Story injections (John):** Make the 4 major-concern story additions (~12 additional pts; ~1 sprint at current velocity for the gap-closing pass).
3. **Architectural addenda (Winston):** Append Concern #5 (`tenant.provisioned` event) to architecture amendment §5.3; address Concern #7 (N8N template source-of-truth) — recommend Option (a) commit JSON exports.
4. **Re-run readiness:** After the above, re-run `bmad-check-implementation-readiness` against the updated artefact set. Verdict should transition to PASS.
5. **Operational follow-up (out of scope for orchestrator):**
   - Confirm SirmaAI EU residency in writing (launch-blocking per Concern #6 history + arch §11.3 risk #1).
   - Get SirmaAI per-Org/per-Project idle cost number (for Concern #10 product decision).

### Sign-off for Orchestrator Sprint Replan

**❌ ~~Not yet cleared for orchestrator dispatch.~~**

**✅ CLEARED FOR ORCHESTRATOR SPRINT REPLAN (gap-closing pass complete, 2026-05-12).**

All 10 concerns from the initial pass have been resolved via targeted story injections, AC tightening, architectural addenda, and one product decision. See **§Gap Closure Pass — 2026-05-12** below for resolution audit.

Per `project_sprint_status_file.md` memory rule: **`sprint-status.yaml` is orchestrator-managed — surgical edits only.** This readiness report does NOT itself modify sprint-status; it authorises the orchestrator to do so now that the gap-closing pass is complete.

### Final Note

This assessment identified **10 issues across 4 categories** on the initial pass (coverage gap, schema migration ownership, ingress configuration, scope-substitution backfill). All 10 are now closed — 4 majors via story injections, 6 minors via AC tightening and architectural addenda. The decision tree, ADR coherence, dependency chain, ACs, and traceability matrix are sound.

Gap-closing effort actual: **<1 day** (faster than the 2-3 day estimate because the gaps were each well-bounded; no architectural rework was needed and the product call landed in a single round-trip).

---

**Author:** 📋 John (PM) — 2026-05-12
**Initial verdict:** CONCERNS — gap-closing pass required (10 gaps)
**Final verdict:** ✅ **PASS** — cleared for orchestrator sprint replan

---

## Gap Closure Pass — 2026-05-12

The following resolves all 10 concerns from the initial pass.

### 🔴 Major Concerns (all 4 resolved)

| # | Original Gap | Resolution | Owning Story / Location |
|---|---|---|---|
| 1 | NFR-25 (tier→rate-limit sync) unowned | Injected story; subscribes to `subscription.changed`, calls `PATCH /api/admin/organizations/{orgId}/rate-limit` within 60s SLO; tier-mapping table externalized to `config/tier_rate_limits.yaml` for commercial calibration | **E04 amendment S04.28** (3 pts) |
| 2 | 4 schema migrations had no explicit owning story | Expanded existing migration stories. `gateway.webhook_subscriptions` + `gateway.workflow_runs` → S04.21 expanded scope. `client.sirmaai_mcp_servers` → S24.01 expanded scope. `gateway.webhook_dlq` → S28.06 expanded scope. No new stories needed; existing story descriptions enlarged | **E04 S04.21**, **E24 S24.01**, **E28 S28.06** (~3 pts scope total) |
| 4 | Public ingress for webhook receiver contradicted original E04 AC | Injected ingress story with manual nginx on www1 per `project_deploy_nginx_manual.md`; runbook entry at `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`; AC tightened to clarify webhook receiver path is publicly exposed while all other gateway paths remain ClusterIP-only | **E04 amendment S04.29** (3 pts) |
| 6 | No backfill story for pre-pivot existing companies | Injected one-off backfill task with pre-flight check, throttle (1/min), idempotent resume; admin endpoint `POST /api/v1/admin/sirmaai-projects/backfill`; runbook at `eusolicit-docs/runbooks/sirmaai-backfill.md` | **E24 S24.08** (3 pts) |

**Major-concern scope addition:** ~12 pts (matches initial estimate).

### 🟠 Minor Concerns (all 6 resolved)

| # | Original Gap | Resolution | Location |
|---|---|---|---|
| 3 | `eusolicit-kraftdata` → `eusolicit-sirmaai` package rename + consumer-service updates lacked explicit story | Injected explicit rename + consumer-pyproject + import-rewrite story | **E04 amendment S04.30** (5 pts) |
| 5 | `tenant.provisioned` event missing from Event Catalog §5.3 | Appended `tenant.provisioned` + `sirmaai.key_rotated` to architecture amendment §5.3 internal events list | **arch amendment §5.3** (addendum) |
| 7 | N8N workflow template source-of-truth not specified | Decided: **commit JSON exports** to `eusolicit-app/infra/n8n-templates/<name>-v<semver>.json` as canonical; deploy script `scripts/sync_n8n_templates.py` diffs and applies on deploy; refuses on drift unless `--force`. Owning story expanded | **arch amendment §3.4 addendum**, **E05 amendment S05.20 expanded scope** (+2 pts) |
| 8 | Local-dev SirmaAI story unowned | Injected explicit local-dev configuration story: point at SirmaAI staging via `SIRMAAI_BASE_URL` env override + shared dev Org/Project; documented in `eusolicit-app/CLAUDE.md`; `sirmaai-mock` deferred per ADR-019 | **E04 amendment S04.31** (2 pts) |
| 9 | `shared.audit_log.details JSONB` accommodation for `erasure_step` not verified | Tightened AC in S24.07 to require `EXPLAIN ANALYZE` verification + GIN-or-expression-index on `details->>'erasure_step'` if seq-scan detected | **E24 S24.07** (AC tightened) |
| 10 | Free-tier Project policy unmodeled — product decision pending | **Deb decision 2026-05-12:** Free tier follows standard provisioning flow; rate limit set by NFR-25 tier mapping (100 runs/day for Free). No tier-conditional code path in E24; tier behavior is rate-limit-driven only. Documented as explicit AC in E24 | **E24 AC additions** |

**Minor-concern scope addition:** ~9 pts (S04.30 5 + S04.31 2 + S05.20 expansion 2).

### Total amendment-effort delta after gap-closure

| Epic | Pre-gap-closure pts | Post-gap-closure pts | Delta |
|---|---|---|---|
| E04 amendment | 23 | 36 | +13 (S04.21 +2, S04.28 +3, S04.29 +3, S04.30 +5, S04.31 +2; –2 net rebalance from S04.27 staying single-purpose) |
| E05 amendment | 26 | 28 | +2 (S05.20 deploy mechanism scope expansion) |
| E11 amendment | 16 | 16 | 0 |
| E17 amendment | 31 | 31 | 0 |
| E24 | 21 | 24 | +3 (S24.08 backfill) |
| E25 | 34 | 34 | 0 |
| E26 | 34 | 34 | 0 |
| E27 | (shared w/ E17) | (shared w/ E17) | 0 |
| E28 | 21 | 22 | +1 (S28.06 migration scope expansion) |
| **TOTAL** | **~206** | **~225** | **+19 pts** |

Adds ~1 sprint of pre-launch effort. Launch slip estimate updates from 8-10 weeks to **9-11 weeks** from approval.

### Verification

After the gap-closing pass, re-ran the FR/NFR coverage matrix, schema migration ownership table, and cross-cutting invariant validation. **All checks pass.** No new gaps surfaced.

| Check category | Status |
|---|---|
| FR/NFR coverage (18 of 18) | ✅ PASS (was 17 of 18) |
| Schema migrations (7 of 7 tables) | ✅ PASS (was 4 of 7) |
| Cross-cutting invariants (10 of 10) | ✅ PASS (unchanged — these were already sound) |
| Epic quality (user-value, independence, AC quality) | ✅ PASS (unchanged) |
| UX alignment | ✅ PASS (unchanged) |
| Dependency chain (no cycles, no forward-refs) | ✅ PASS (unchanged) |

### Operational follow-up (still required, out of scope for orchestrator)

These items do not block orchestrator dispatch but must be tracked to launch:

1. **Confirm SirmaAI EU data residency in writing** (launch-blocking per arch §11.3 risk #1). Contract item with SirmaAI; not a planning-artifact gap.
2. **Get SirmaAI per-Org/per-Project idle cost number.** Used to validate the Free-tier policy choice (Concern #10). If idle cost is materially higher than expected, Topology A → Topology C (hybrid by tier) revisit may be warranted post-launch.
3. **Tier-rate-limit mapping commercial calibration.** The numbers in S04.28 (Free 100/day, Starter 1k/day, Professional 10k/day, Pro+ 20k/day, Enterprise 100k/day) are technical defaults; commercial team needs to confirm against pricing model before E04 amendment ships.

---

**Re-readiness verdict: ✅ PASS — orchestrator may proceed with sprint replan.**
