---
status: "Draft — PM-authored under bmad-agent-pm; intended as the single source of truth for build dispatch sequencing post-IR remediation"
author: "📋 John (PM)"
date: "2026-05-15"
inputs:
  - "eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-15.md"
  - "eusolicit-docs/planning-artifacts/PRD.md + prd-amendment-2026-05-12-sirmaai.md"
  - "eusolicit-docs/planning-artifacts/architecture.md + architecture-amendment-2026-05-12-sirmaai.md"
  - "eusolicit-docs/planning-artifacts/epics/E01–E28*.md"
  - "eusolicit-docs/planning-artifacts/ux-spec.md + ux-spec-amendment-2026-05-15-sirmaai.md"
  - "eusolicit-docs/implementation-artifacts/sprint-status.yaml"
  - "eusolicit-docs/implementation-artifacts/pm-audit-memo-e22-e23-dispatch-2026-05-15.md"
audience: "Orchestrator (BMAD dispatch) + dev pool + operator + PM"
---

# EU Solicit — Build Sequence & Schedule (2026-05-15)

> **Reading order:** start at §1; if you only have 5 minutes, read §1, §2, §6, §11. The schedule (§6) is the load-bearing section.

## 1. Purpose

This document is the **single source of dispatch truth** for everything between today (2026-05-15) and the SirmaAI-feature-complete milestone (~2026-08-15). It:

- Catalogues every open story across all epics in dependency order.
- Slices the backlog by **vertical user-journey**, not horizontal layer, so each sprint produces a complete value moment instead of a stack of half-features.
- Front-loads technical dependencies so they don't block user-facing work mid-sprint.
- Gives sprint-by-sprint dispatch sequencing with explicit ready-to-start gates per story.
- Identifies the critical path + the risks that could move it.

This document does NOT replace the orchestrator's sprint-status.yaml or any epic file. It is the *plan*; those are the *state*. When they disagree, sprint-status.yaml is canonical for execution state and this document is canonical for intent.

## 2. Current state snapshot (verified on-disk 2026-05-15)

### Done (no further action)
- **All Phase-1 legacy epics**: E01–E21 done with retrospectives.
- **E04 SirmaAI-amendment substrate**: S04.21–S04.31 done; only **S04.20 (sirmaai-gateway rename)** remains in-progress.
- **E05 N8N data-pipeline amendment**: S05.20–S05.23 done; **S05.24 in-progress**; S05.25 backlog.
- **E13 (Hardening) carry-forwards**: drift-recovery + inj-03 closed 2026-05-13.

### In flight (today's "work pool")
- **E04 S04.20** — `sirmaai-gateway` rename + flag scaffold. **Critical path.** Every downstream SirmaAI epic implicitly depends on it.
- **E05 S05.24** — N8N staged-rollout enforcement. Critical for safe pipeline cutover.
- **E22 onprem-01..06** — 6 stories at `review`, awaiting bmad-code-review Approve cascade + operator drills.
- **E23 pe-04 + public-sla** — at `ready-for-dev`, both unblocked by IR remediation as of 2026-05-15.

### Backlog (planning-complete; ready for orchestrator pickup)
- **E05 S05.25** (cross-tenant negative tests)
- **E11 amendment** (S11.20–S11.23): SirmaAI Project template seed, Regulation Tracker retire, E11 endpoint swap, KB-grounded ESPD tests
- **E17 amendment / E27** (S17.30a, S17.30b, S17.31–S17.36 + S27.08): CRM via MCP, Dynamics 365 + HubSpot
- **E22 onprem-07** (NEW 2026-05-15): N8N instance ops
- **E23 sirmaai-eu-residency-due-diligence** (NEW 2026-05-15): legal/operator track
- **E24 S24.01–S24.08**: SirmaAI tenant provisioning
- **E25 S25.01–S25.09**: KB lifecycle
- **E26 S26.01–S26.10**: agent-driven ingestion
- **E28 S28.01–S28.08**: webhook + reconciler hardening

**Total backlog work**: ~50 stories, ~250–300 story points.

## 3. Definition of Done — per slice

Each vertical slice has its own DoD. Crossing the DoD line is what determines whether the slice is "ship-ready" — independent of any downstream slice.

| Slice | DoD signal |
|---|---|
| Slice 0 (launch as-is) | E22+E23 all done; S04.20 done; product accessible at production URL with the existing legacy KraftData→SirmaAI route working; Trust Center disclosure live; 2-week soak gate begins |
| Slice 1 (SirmaAI tenant substrate) | Every new EU Solicit company auto-provisions a SirmaAI Project; archive flow works end-to-end; nightly reconciliation runs; admin orphan-list UI surfaces failures |
| Slice 2 (KB lifecycle for users) | Users upload artefacts to their KB via UI; processing badges flip; semantic search works; tier-gated quota enforced; agents (existing AND new) ground in KB |
| Slice 3 (Qualification + Quantification) | Auto-qualification fires on every ingested opportunity; Pro+ users see quantification on `pursue` outcomes; citation chips link back to KB sources; tier-gated correctly |
| Slice 4 (CRM via MCP) | Pro+ users OAuth-connect Dynamics 365 + HubSpot; qualification agents enrich via MCP tools; conflict log surfaces LWW resolutions; token rotation Beat operational |
| Slice 5 (Hardening + close) | Webhook DLQ admin surface; HMAC rotation Beat operational; degraded-mode banner shipping; tenant cross-tenant negative tests added across all SirmaAI epics; E11 amendment closes (KB-grounded ESPD) |

Each slice is **releasable independently**. Sprint 5 ends with the full SirmaAI feature set live and stable.

## 4. Slicing strategy — why vertical, not horizontal

A horizontal slicing strategy ("build all the database schemas, then all the APIs, then all the UIs") would have us shipping no user value for 4–6 weeks while we built backend plumbing. A vertical strategy slices through every layer for one user-visible capability at a time.

The trade-off:
- ✅ Vertical: shippable value every 2–3 weeks; risk-of-rework lower (we discover integration issues at the time the integration is built, not at end-of-project); morale-positive (each sprint shows real product).
- ⚠️ Vertical: each slice carries the cost of "all layers" for a smaller scope — schema, API, UI, tests — so the early slices feel slower per-feature than a horizontal plan would. Worth it because the *cycle time per delivered capability* is faster overall.

**Operating rule:** within each slice, technical dependencies are addressed FIRST inside that slice (schema → service → API → UI → tests). Across slices, the substrate (Slice 1) precedes value features (Slice 2+). This is the "spine first, then features" approach popularised by Marty Cagan + Teresa Torres.

## 5. Inventory & dependency map

### 5.1 Dependency graph (epic-level)

```
                                  ┌─────────────────┐
                                  │  S04.20 rename  │  (critical path — blocks every SirmaAI epic)
                                  └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │   E24 (8 sty)   │  Tenant provisioning + Project lifecycle
                                  └────────┬────────┘
                                           │
                       ┌───────────────────┼───────────────────┐
                       ▼                   ▼                   ▼
                ┌────────────┐      ┌────────────┐      ┌────────────┐
                │ E25 (9 sty)│      │ E28 (8 sty)│      │ E11 amend  │
                │  KB lcycle │      │ Webhk hard │      │ ESPD+grant │
                └─────┬──────┘      └─────┬──────┘      └────────────┘
                      │                   │                   ▲
                      └────────┬──────────┘                   │
                               ▼                              │
                       ┌──────────────────┐                   │
                       │ E26 (10 stories) │───────────────────┘  (E11 amend depends on Project template seed in E24+E26)
                       │ Qualifier/Quanti │
                       └──────────┬───────┘
                                  │
                                  ▼
                         ┌──────────────────┐
                         │ E17/E27 (8 sty)  │  CRM via MCP (D365 + HubSpot)
                         │ S17.30a..36 + 08 │
                         └──────────────────┘
```

Parallel tracks (NOT on critical path):
- **E22 onprem-01..07** — parallel with everything; needed for launch posture (Slice 0)
- **E23 pe-04 + public-sla + residency** — parallel with everything; needed for launch posture (Slice 0)
- **E05 S05.24 + S05.25** — closes the N8N cutover; parallel with E24 if S04.20 done

### 5.2 Per-story dependency table (backlog only)

Format: story key — pts — depends-on — blocks

**E04 amendment closure:**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| 4-20-service-rename-and-flag-scaffold | (in-flight) | — | EVERY SirmaAI story below |

**E05 amendment closure:**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| 5-24-n8n-staged-rollout-enforcement | (in-flight) | S05.20–S05.23 done | safe ingest cutover |
| 5-25-tenant-aware-cross-tenant-negative-tests | 3 | S05.24 done; E24 schema | none (test layer) |

**E22 closure (Slice 0 launch posture):**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| onprem-01–06 (review→done) | — | bmad-code-review Approve + operator drill | launch readiness gate |
| onprem-07-n8n-instance-ops | 5 | onprem-03 (observability stack) | post-launch N8N upgrade safety |

**E23 closure (Slice 0 launch posture):**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| pe-04-chaos-drill-execution | 3 (drill work) | onprem-03 + onprem-04 | launch readiness gate |
| public-sla-announcement-soak-gate | 1 | i18n ✓ already; needs Approve + legal sign-off | launch readiness gate |
| sirmaai-eu-residency-due-diligence | 1 (PM+legal) | SirmaAI commercial cycle | **launch readiness gate (PRD-amended)** |

**E24 SirmaAI tenant provisioning:**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| 24-01-sirmaai-projects-schema-and-state-machine | 3 | S04.20 done | every other E24 story |
| 24-02-sirmaai-project-creation-on-company-create | 5 | 24-01 | 24-03..06 |
| 24-03-sirmaai-project-template-seed | 5 | 24-02 + E11 S11.20 + E26 agent defs | every agent path |
| 24-04-tenant-default-mcp-server-registrations | 3 | 24-03; S17.30a+S17.31 specs landed | E17/E27 OAuth flow |
| 24-05-nightly-reconciliation-and-admin-reprovision | 3 | 24-02 | none (test + ops surface) |
| 24-06-company-archive-sirmaai-project-soft-delete | 2 | 24-02 + 24-04 | 24-07 |
| 24-07-right-to-erasure-cross-substrate-completion-proof | 3 | 24-06 + 25-01 (KB schema) | FR-56 closure |
| 24-08-one-off-backfill-provision-for-existing-companies | 3 | 24-01..06 | post-cutover operability |

**E25 Knowledge Base lifecycle:**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| 25-01-sirmaai-kb-files-schema-and-upload-metadata | 3 | 24-01 (schema migration ordering) | every other E25 story |
| 25-02-upload-proxy-and-storage-resource-integration | 5 | 25-01 + 24-02 (Project exists) | 25-04, 25-07 |
| 25-03-storage-processed-webhook-handler-and-ui-state | 3 | 25-02 + 28-01 (subscriptions) | UI completeness |
| 25-04-semantic-search-parsed-text-download-proxies | 5 | 25-02 | 25-09, 26-08 |
| 25-05-replace-archive-and-restore-lifecycle | 3 | 25-02 | 24-07 (erasure) |
| 25-06-profile-update-kb-re-index | 2 | 25-02 + 25-05 | none |
| 25-07-tier-gated-quota-and-workspace-kb-dashboard | 5 | 25-02 + 25-04 | UI completeness |
| 25-08-nightly-sha-256-reconciliation | 3 | 25-02 | none (data-integrity layer) |
| 25-09-agent-grounding-and-citation-surface | 5 | 25-04 + 26-08 (citation UI) | DoD for Slice 2/3 |

**E26 Agent-driven ingestion:**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| 26-01-opportunity-qualifier-sirmaai-agent-definition | 3 | E26 baseline spec (NEW 2026-05-15) curated; 24-03 template | 26-03 |
| 26-02-opportunity-quantifier-sirmaai-agent-definition | 3 | E26 baseline spec curated; 24-03 template | 26-03 |
| 26-03-opportunity-analysis-v1-n8n-workflow-template | 5 | 26-01 + 26-02 + S05.24 done | 26-05, 26-06 |
| 26-04-opportunity-analyses-schema-and-service-layer | 3 | 24-01 | 26-05, 26-07, 26-08 |
| 26-05-user-trigger-endpoint-opportunities-analyze | 3 | 26-03 + 26-04 | 26-08 UI |
| 26-06-auto-trigger-on-opportunities-ingested | 3 | 26-03 + 26-04 + S05.24 done | end-to-end qual cascade |
| 26-07-webhook-driven-result-persistence-and-sse-notification | 3 | 26-04 + 28-03 (receiver hardening) | 26-08 |
| 26-08-opportunity-detail-ui-for-analysis-results | 5 | 26-04 + 26-07 + ux-amendment §3-J1 | DoD for Slice 3 |
| 26-09-kb-grounding-regression-tests-and-structured-output-validation | 3 | 26-01..03 | quality gate |
| 26-10-equivalence-harness-vs-legacy-e11-quantification | 3 | 26-03 + legacy E11 still live | safe cutover |

**E17 amendment / E27 CRM via MCP:**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| 17-30a-dynamics-365-mcp-server-tool-spec-authoring | 4 | (planning-only — can start with S04.20 done) | 17-30b, 24-04 |
| 17-30b-dynamics-365-mcp-server-registration-flow-and-sandbox-tests | 4 | 17-30a + 24-04 | tenant onboarding for D365 |
| 17-31-hubspot-mcp-server-tool-spec-registration | 5 | (parallel with 17-30a/b) + 24-04 | tenant onboarding for HubSpot |
| 17-32-oauth-callback-hosting-and-fernet-token-vault | 5 | 17-30b + 17-31 (specs ready) | 17-33, 17-34 |
| 17-33-token-rotation-double-sided | 5 | 17-32 | safe long-term ops |
| 17-34-mcp-tool-invocation-log-and-conflict-resolution | 3 | 17-32 | qual+quant CRM enrichment |
| 17-35-tier-gate-pro-plus-and-workspace-crm-dashboard | 3 | 17-32 + 17-34 + ux-amendment §5.14 | DoD for Slice 4 |
| 17-36-workspace-archive-mcp-secret-deletion | 2 | 17-32 + 24-06 | 24-07 (erasure) |
| 27-08-agent-side-mcp-tool-consumption-patterns | 3 | 17-30b + 17-31 + 26-02 | quality gate for CRM-enriched quant |

**E28 Webhook & Reconciler Hardening:**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| 28-01-subscription-bootstrap-and-lifecycle-management | 3 | S04.20 done | every webhook consumer |
| 28-02-hmac-secret-rotation-celery-beat | 3 | 28-01 | safe long-term webhook ops |
| 28-03-webhook-receiver-hardening-and-idempotency-cache | 5 | (E04 S04.25 scaffold already done) + 28-01 | 25-03, 26-07 |
| 28-04-internal-event-routing | 3 | 28-03 | all webhook consumers |
| 28-05-reconciler-hardening-and-observability | 3 | (E04 S04.26 scaffold already done) | DoD for run-state integrity |
| 28-06-webhook-dlq-migration-and-admin-surface | 3 | 28-03 | operator playbook for poison events |
| 28-07-tenant-visible-degraded-mode-banner | 2 | 28-05 + ux-amendment §5.13 | DoD for Slice 5 |
| 28-08-cross-tenant-negative-and-idempotency-rule-tests | 2 | 28-01..07 | quality gate |

**E11 amendment:**
| Story | Pts | Depends on | Blocks |
|---|---|---|---|
| 11-20-sirmaai-project-template-seed-grant-compliance-agents | 5 | 24-03 (template seed mechanism) | 11-21, 11-22 |
| 11-21-regulation-tracker-n8n-workflow-celery-retire | 5 | 11-20 + S05.24 done | safe Celery retire |
| 11-22-e11-endpoint-internal-call-path-swap | 3 | 11-20 | DoD for Slice 5 |
| 11-23-kb-grounded-espd-and-grant-tests | 3 | 11-20..22 + 25-09 | quality gate |

### 5.3 Critical-path stories

The shortest possible path to **all SirmaAI features live** is:

`S04.20 → 24-01 → 24-02 → 24-03 → 26-01+26-02+26-04 → 26-03 → 26-05+26-06+26-07 → 26-08`

That's **10 sequential stories**. Every other story can be parallelised around this path.

## 6. Sprint-by-sprint schedule

**Sprint cadence:** 2 weeks each. Sprint 0 starts today (2026-05-15). Sprint 5 ends 2026-08-21.

### Sprint 0 — Launch gate clearance (2026-05-15 → 2026-05-29) — Slice 0

**Goal:** clear the 2026-06-01 launch gate. Zero new SirmaAI features.

**Dispatch parallel:**

Track A — **E22 onprem cascade**:
1. `bmad-code-review` on onprem-01 (sequence-first)
2. Parallel `bmad-code-review` on onprem-02..06
3. Operator drill: full restore (onprem-01); controlled-restart (onprem-02); monitoring decision (onprem-03); first synthetic page (onprem-04); branch-protection visual (onprem-05); sacrificial VM rebuild (onprem-06)
4. Each → done; E22 epic flips in-progress → done

Track B — **E23 close**:
1. `bmad-code-review` on pe-04 chaos runbook (now includes full Drill 4 + Drill 5 + Drill 6 per IR remediation 2026-05-15)
2. Operator: execute 6 drills, populate §Drill Results, post-mortems
3. `bmad-code-review` on public-sla (i18n ✓ verified 2026-05-15)
4. Product + legal sign-off on disclosure copy
5. Initiate sirmaai-eu-residency-due-diligence (PM + legal cycle starts NOW — long-pole)
6. pe-06 reconciliation: operator confirms alert-delivery test result on www1; append §Verification screenshot

Track C — **E04 S04.20 closure**:
1. Dev pool completes the gateway rename
2. `bmad-code-review` Approve
3. Story → done — **unblocks Sprint 1**

Track D (PARALLEL, no blocker on launch): **E22 onprem-07 NEW story** stays in backlog this sprint; defer to Sprint 1 unless operator capacity allows.

**Sprint 0 exit criteria:**
- E22 epic done (6/6 stories + N8N AC deferred)
- E23 epic done OR residency story is the only open item (deferred per legal cycle)
- S04.20 done
- Trust Center disclosure live
- Product accessible at production URL
- 2-week soak gate begins

**Risk:** if residency legal cycle takes >2 weeks, launch posture must explicitly carry "SirmaAI sub-processor residency confirmation pending — will publish on Trust Center on signing" as a known acceptable risk. Decision-owner: Deb.

**Total points in Sprint 0:** ~10 story-points of net-new code (mostly review + drill work) + 1 PM-track legal item.

---

### Sprint 1 — SirmaAI tenant substrate (2026-05-29 → 2026-06-12) — Slice 1

**Goal:** every new EU Solicit company auto-provisions a SirmaAI Project. Existing legacy KraftData-route AI features continue to work unchanged (backwards compat).

**Dispatch sequence:**

1. **24-01** — schema + state machine (foundation; 3pts, backend)
2. Parallel: **17-30a** + **17-30b** — Dynamics 365 MCP spec authoring (4+4pts, parallel-trackable with E24 because no overlap)
3. **24-02** — Project creation on company create (5pts; depends on 24-01)
4. **17-31** — HubSpot MCP spec + registration (5pts, parallel-trackable with 24-02; sandbox-tests deferrable to Sprint 2)
5. **28-01** — webhook subscription bootstrap (3pts; depends on S04.20 only — parallel with 24-01)
6. **28-03** — webhook receiver hardening (5pts; parallel with 24-02; closes idempotency-cache gap)
7. **24-05** — nightly reconciliation + admin reprovision (3pts; depends on 24-02; **enables admin orphan-recovery from day one** — critical for Slice 1 DoD)
8. **5-25** — cross-tenant negative tests for N8N pipeline (3pts; closes E05 amendment; parallel)

**Sprint 1 exit criteria (Slice 1 DoD):**
- New companies auto-provision SirmaAI Projects (no KB / agent seed yet — that's Sprint 2)
- Admin orphan list works
- Webhook subscription registered + HMAC verified + DLQ scaffold live
- E04 amendment fully done (S04.20 closed Sprint 0 + S04.21..31 already done)
- E28 partially done (28-01 + 28-03)

**Total points Sprint 1:** ~32

---

### Sprint 2 — KB lifecycle + Project template seed (2026-06-12 → 2026-06-26) — Slice 2 (start)

**Goal:** Elena can upload past proposals + ESPD templates to KB; agents (existing and new) can ground in KB.

**Dispatch sequence:**

1. **25-01** — KB files schema + upload metadata model (3pts; depends on 24-01 schema isolation pattern)
2. **24-03** — Project template seed (5pts; depends on 24-02; needs 11-20 SirmaAI Project template seed for grant/compliance agents AS WELL AS 26-01/02 qualifier+quantifier definitions — see footnote below)
3. **11-20** — SirmaAI Project template seed for grant/compliance agents (5pts; PARALLEL with 24-03 because both feed into Project template — coordinate via agent-config-repo PR review)
4. **26-01** — opportunity_qualifier agent definition + Project template addition (3pts; **prerequisite: E26 baseline dataset spec curation completes** — owned by Murat TEA; PM coordinates SME hire 2026-05-29 onwards)
5. **26-02** — opportunity_quantifier agent definition + Project template addition (3pts; same prerequisite as 26-01)
6. **25-02** — KB upload proxy + storage-resource integration (5pts; depends on 25-01 + 24-02)
7. **24-04** — Tenant-default MCP server registrations (3pts; depends on 24-03 + 17-30a/b + 17-31 specs ready; injects Dynamics + HubSpot stubs)
8. **25-07** — Tier-gated quota + workspace KB dashboard (5pts; depends on 25-02; Slice 2 UI surface)

**Footnote on 24-03 dependency:** the Project template binds agent definitions (qualifier, quantifier, ESPD, grant, compliance) at seed time. If 24-03 lands before 11-20 + 26-01 + 26-02, those agents simply aren't seeded yet — that's OK, template-version-bump in Sprint 3 adds them. Trade-off: fewer agents in early-cohort tenant Projects vs. waiting until ALL agent defs are ready. Recommend: ship 24-03 in Sprint 2 with a minimal set (just the legacy `executive_summary` + `proposal_drafter` paths), then template-version-bump to v2 in Sprint 3 once qualifier/quantifier/ESPD/grant defs are ready.

**Sprint 2 exit criteria (Slice 2 start):**
- KB upload works end-to-end (file → SirmaAI storage-resource → DB row → processing-badge UI)
- Tier-gated quota meter live (UI + backend enforcement)
- Project template seeded with legacy agent defs
- Qualifier + quantifier agent definitions authored + eval-runs pass ≥85% baseline alignment

**Total points Sprint 2:** ~32

---

### Sprint 3 — Qualification + Quantification + KB completeness (2026-06-26 → 2026-07-10) — Slice 2 (finish) + Slice 3

**Goal:** automated qualification on every ingested opportunity; user-triggered quantification on `pursue`; citation chips link back to KB; KB lifecycle complete.

**Dispatch sequence:**

1. **26-04** — opportunity_analyses schema + service layer (3pts; depends on 24-01)
2. **25-03** — storage-processed webhook handler + UI state badge (3pts; depends on 25-02 + 28-01)
3. **26-03** — opportunity-analysis-v1 N8N workflow template (5pts; depends on 26-01 + 26-02 + S05.24 done)
4. **25-04** — Semantic search + parsed-text + download proxies (5pts; depends on 25-02)
5. **25-05** — Replace + archive + restore lifecycle (3pts; depends on 25-02)
6. **25-06** — Profile-update KB re-index (2pts; depends on 25-05)
7. **26-05** — User-trigger endpoint (3pts; depends on 26-03 + 26-04)
8. **26-06** — Auto-trigger on opportunities.ingested (3pts; depends on 26-03 + 26-04 + S05.24)
9. **26-07** — Webhook-driven persistence + SSE notification (3pts; depends on 26-04 + 28-03)
10. **26-08** — Opportunity-detail UI for analysis results (5pts; depends on 26-04 + 26-07 + ux-amendment §3.J1)
11. **25-09** — Agent grounding + citation surface (5pts; depends on 25-04 + 26-08)

**Sprint 3 exit criteria (Slice 2 + Slice 3 DoD):**
- New opportunity ingestion → auto-qualification → UI shows result < 60s steady-state
- Pro+ user clicks "Run quantification" → result with citations < 3 min
- Semantic KB search works end-to-end
- Replace/archive/restore lifecycle UI live
- Profile update re-indexes KB within 5 min

**Total points Sprint 3:** ~40 (heaviest sprint; recommend splitting some stories across into Sprint 4 if velocity sub-target)

---

### Sprint 4 — CRM via MCP + reconciler hardening (2026-07-10 → 2026-07-24) — Slice 4

**Goal:** Pro+ users OAuth-connect Dynamics 365 + HubSpot; agents enrich opportunities via MCP tools.

**Dispatch sequence:**

1. **17-32** — OAuth callback hosting + Fernet token vault + SirmaAI secret push (5pts; depends on 17-30b + 17-31)
2. **17-34** — MCP-tool invocation log + conflict resolution (3pts; depends on 17-32)
3. **17-33** — Token rotation double-sided Celery Beat (5pts; depends on 17-32)
4. **17-36** — Workspace archive → MCP secret deletion (2pts; depends on 17-32 + 24-06)
5. **24-06** — Company archive → SirmaAI Project soft-delete + key revocation (2pts; depends on 24-02 + 24-04)
6. **24-07** — Right-to-erasure cross-substrate completion proof (3pts; depends on 24-06 + 25-01..05 done; closes FR-46 + FR-56)
7. **28-05** — Reconciler hardening + observability (3pts; depends on existing S04.26 scaffold)
8. **28-06** — Webhook DLQ migration + admin surface (3pts; depends on 28-03)
9. **17-35** — Tier-gate Pro+ + workspace CRM dashboard widget (3pts; depends on 17-32 + 17-34 + ux-amendment §5.14)
10. **27-08** — Agent-side MCP-tool consumption patterns (3pts; depends on 17-30b + 17-31 + 26-02)

**Sprint 4 exit criteria (Slice 4 DoD):**
- Pro+ user connects D365 in < 2 min via OAuth
- Pro+ user connects HubSpot same flow
- Qualification + quantification agents call MCP tools mid-run (verified via 27-08 eval-runs)
- Token rotation Beat runs successfully (verified once on staging)
- Workspace archive cleanly tears down cross-substrate state
- Reconciler observable in Grafana

**Total points Sprint 4:** ~32

---

### Sprint 5 — Hardening + polish + E11 amendment close (2026-07-24 → 2026-08-07) — Slice 5

**Goal:** all observability, defence-in-depth, and tail-end stories close. Platform feature-complete for the SirmaAI scope.

**Dispatch sequence:**

1. **28-02** — HMAC secret rotation Celery Beat (3pts; depends on 28-01)
2. **28-04** — Internal event routing (3pts; depends on 28-03)
3. **28-07** — Tenant-visible degraded-mode banner (2pts; depends on 28-05 + ux-amendment §5.13)
4. **28-08** — Cross-tenant negative + idempotency-rule tests (2pts; depends on 28-01..07)
5. **24-08** — One-off backfill provision for existing companies (3pts; depends on 24-01..06; **execute once on prod after Sprint 5 lands**)
6. **25-08** — Nightly SHA-256 reconciliation (3pts; depends on 25-02)
7. **26-09** — KB-grounding regression tests + structured-output validation (3pts; depends on 26-01..03)
8. **26-10** — Equivalence harness vs legacy E11 quantification (3pts; depends on 26-03; required before retiring legacy E11)
9. **11-21** — Regulation Tracker N8N workflow + Celery retire (5pts; depends on 11-20 + S05.24)
10. **11-22** — E11 endpoint internal call-path swap (3pts; depends on 11-20)
11. **11-23** — KB-grounded ESPD + grant tests (3pts; depends on 11-20..22 + 25-09)

**Sprint 5 exit criteria (Slice 5 DoD = full SirmaAI feature-complete):**
- All E11 amendment stories done; legacy KraftData ESPD path retired
- All E24–E28 stories done
- All E17 amendment / E27 stories done
- Hardening complete: rotation Beats running, DLQ admin surface live, banner shipping
- Cross-tenant negative tests passing across every new schema
- Backfill executed on production company table

**Total points Sprint 5:** ~33

---

### Sprint 6 — Retrospectives + post-launch polish (2026-08-07 → 2026-08-21) — Reserve

**Goal:** retros for all SirmaAI epics; project-context.md updates with patterns/anti-patterns learned; baseline-dataset v2 (post-launch SME feedback incorporated); any defects/follow-ups surfaced.

**Recommended activities:**
- `bmad-retrospective` per SirmaAI epic (E24, E25, E26, E27, E28, E11 amendment)
- Update `project-context.md` with E24..E28 pattern/anti-pattern entries
- Murat (TEA): adversarial review on the qualifier + quantifier agents at production scale; baseline-dataset v2 curation
- Operator: 60-day soak with full SirmaAI scope live; gather data toward future numeric SLA decision
- Reserve: ~20% of sprint capacity for defects + post-launch follow-ups discovered during Sprint 5

---

## 7. Critical path summary

Strictest dependency chain (sprint-by-sprint key stories):

```
Sprint 0:  S04.20  →  E22 done  →  E23 done (residency may slip)
Sprint 1:  24-01   →  24-02   →  24-05   →  28-01  →  28-03
Sprint 2:  24-03   →  26-01 + 26-02 + 11-20  →  25-01 + 25-02
Sprint 3:  26-04   →  26-03   →  26-05 + 26-06 + 26-07  →  26-08 + 25-09
Sprint 4:  17-32   →  17-33/34/35  →  24-06   →  24-07
Sprint 5:  28-02   →  28-04/07/08  →  11-21/22/23
```

**Critical-path story count:** ~18 sequential stories.
**Critical-path risk:** if 24-03 (Project template seed) slips, **Sprint 3 cannot start** because qualifier + quantifier need to be in the template. Mitigation: ship 24-03 with minimal template in Sprint 2 + template-version-bump in Sprint 3. (Already documented in §6 Sprint 2 footnote.)

## 8. Risk register

| # | Risk | Probability | Impact | Mitigation | Owner |
|---|---|---|---|---|---|
| R1 | SirmaAI residency legal cycle exceeds 2 weeks | Medium | Launch slip | Decision: launch with sub-processor disclosure pending + explicit Trust Center note; OR slip launch date by 1-2 weeks | Deb |
| R2 | S04.20 rename has subtle code-route bugs | Medium | Sprint 1 slips | Dev pool to finish before Sprint 1 starts; thorough cross-service smoke test before Approve | dev pool |
| R3 | Sprint 3 (40 pts) exceeds velocity | High | Slice 3 incomplete by Sprint 3 end | Defer 25-09 + 26-09 to Sprint 4; OR add a developer | PM |
| R4 | E26 baseline dataset curation slips (SME hire) | Medium | Sprint 2 qualifier/quantifier blocked | Start hire conversation 2026-05-29 (Sprint 0 end); fallback to single-rater v1 spec already approved | Deb + Murat |
| R5 | UX amendment requires Sally cycles we can't get | High | Stories using new UX states (25-07, 26-08, 28-07, 17-35) can't reach Approve | PM-authored UX amendment (2026-05-15) is dispatchable as-is; Sally refinement is post-Approve polish | Sally + PM |
| R6 | SirmaAI rate-limit surprises during qualifier auto-trigger cascade | Medium | Production opportunity ingestion stalls | Throttle in 26-06 (per-tenant 100 concurrent) + degraded-mode banner (28-07) + reconciler converges; emergency feature-flag disable per-tenant available | dev pool |
| R7 | Dynamics 365 sandbox access | Medium | 17-30b sandbox tests blocked | Procure Dynamics 365 trial early (Sprint 0); HubSpot dev account easier | PM |
| R8 | onprem-07 N8N drill consumes operator capacity | Low | Sprint 0 onprem cascade slips | Defer onprem-07 to Sprint 1 if operator-saturated | PM |
| R9 | Right-to-erasure (24-07) audit-log scan slow | Medium | Performance gate on close | GIN index on audit_log.details (see 24-07 AC + 2026-05-15 readiness Concern #9 carry-forward); EXPLAIN ANALYZE pre-prod | dev pool |
| R10 | N8N major-version upgrade breaks workflow templates | Low | Sprint-2+ ingest at risk | onprem-07 pins version; major-version bumps gated on sprint-change-proposal review | platform-eng |

## 9. Workflow & dispatch conventions

Apply these uniformly throughout Sprints 0–5:

1. **AP17-C1 two-gate close:** every story must pass `bmad-code-review` Approve verdict before flipping `review → done`. PM dev passes ≠ formal story completion (E22-AP01).
2. **AP18-C2 atomic patch:** story file Status header + sprint-status row update in the same commit. No drift.
3. **Schema-first within each story:** stories that include a schema (24-01, 25-01, 26-04) land the migration first; service code consumes the schema, not the other way around.
4. **Cross-tenant negative tests are non-negotiable** on every new API endpoint that touches a `company_id`-scoped resource (project memory rule).
5. **`hmac.compare_digest()` never `==`** for HMAC verification (Rule 48, project memory).
6. **Tier gates via `TierGate Depends`** consistently — no inline checks (ADR-006).
7. **Streaming + async-run UX states** must be wired to SSE per ADR-005 lifecycle invariants; no polling loops.
8. **`Fernet` canonical module** for app-level encryption (Epic 9 canonical) — no ad-hoc crypto.
9. **Operator drills produce real artefacts** (post-mortems, screenshots, §Drill Results table rows) — no synthetic checkmarks.
10. **N8N templates are PR-reviewed code**, not dashboard-edited config — every change goes through repo (semver tags, rollback plan).

## 10. Definition of "ready to start" (per story)

A story is dispatch-ready when ALL of the following hold:

- [ ] Story file exists at `implementation-artifacts/<key>.md`
- [ ] sprint-status row exists for the key
- [ ] All upstream dependencies (per §5.2 table) at `done`
- [ ] Schema migrations the story consumes are at `done` (or the story IS the schema migration)
- [ ] Required UX spec sections in `ux-spec.md` (and/or amendment) exist
- [ ] Required test fixtures (e.g. E26 baseline dataset) exist
- [ ] No critical-path predecessor at `failed` or stuck > 1 sprint

If any condition fails, the story stays in `backlog`. PM coordinates resolution before dispatch.

## 11. TL;DR for the orchestrator

If you only need the next 5 things to dispatch, in order:

1. **bmad-code-review** on `onprem-01-postgres-backup-and-recovery` (E22 sequence-first; everything else in Sprint 0 follows)
2. **bmad-code-review** on `S04.20` once dev pool flags it complete (unblocks every SirmaAI sprint)
3. **bmad-create-story** on `24-01-sirmaai-projects-schema-and-state-machine` (Sprint 1 sequence-first, after S04.20 done)
4. PM action: open SirmaAI commercial conversation for the **DPA addendum** (residency due-diligence is launch-blocking; long-pole; start NOW)
5. PM action: open SME-bid-manager hire conversation for **E26 baseline dataset curation** (Sprint 2 qualifier+quantifier blocker; ~€600 + 2 weeks wall-clock)

Everything else cascades from there per §6 Sprint schedule.

## 12. Open coordination items

These need named owners + dates BEFORE Sprint 1 starts:

- **SirmaAI commercial contact** for DPA addendum: who at SirmaAI does Deb reach? (Risk R1)
- **SME bid manager** for E26 baseline curation: hire path + budget approval (Risk R4)
- **Dynamics 365 sandbox** provisioning: tenant + trial license (Risk R7)
- **Status-page domain** `status.eusolicit.com` provisioned (UX amendment §8 open question 3)
- **Sally (UX)** availability: how many hours/week through Sprints 2–4 for amendment refinement + new screen states authoring as discovered
- **Dispatch capacity**: how many concurrent `bmad-code-review` / `bmad-dev-story` passes can the orchestrator run? Sprint 3 (40 pts) assumes ≥3 concurrent dev tracks
- **Murat (TEA) availability**: he owns 26-09 quality gate + retros + baseline review; need ~25% of his time across Sprints 2–6

## 13. Companion artefacts

- IR report: `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-15.md`
- PM audit memo: `eusolicit-docs/implementation-artifacts/pm-audit-memo-e22-e23-dispatch-2026-05-15.md`
- UX amendment: `eusolicit-docs/planning-artifacts/ux-spec-amendment-2026-05-15-sirmaai.md`
- E26 baseline dataset spec: `eusolicit-docs/test-artifacts/e26-baseline-datasets-spec.md`
- Onprem-07 story: `eusolicit-docs/implementation-artifacts/onprem-07-n8n-instance-ops.md`
- Residency story: `eusolicit-docs/implementation-artifacts/sirmaai-eu-residency-due-diligence.md`
- Chaos drill runbook (now 6 drills): `eusolicit-docs/runbooks/chaos-drill-single-host.md`

---

**Author:** 📋 John (PM) — `bmad-agent-pm` — 2026-05-15
**Maintenance rule:** treat this file as a planning snapshot. As sprint-status.yaml diverges from §5/§6, update this file at sprint boundaries (every 2 weeks) OR archive it under `.archive/build-sequence-2026-05-15.md` and author a fresh snapshot.
