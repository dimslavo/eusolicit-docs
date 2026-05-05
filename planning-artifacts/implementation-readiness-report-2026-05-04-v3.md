---
stepsCompleted:
  - step-01-document-discovery
  - step-02-prd-analysis
  - step-03-epic-coverage-validation
  - step-04-ux-alignment
  - step-05-epic-quality-review
  - step-06-final-assessment
inputDocuments:
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/prd-amendment-2026-04-25.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/ux-spec.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-04-v3.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-04-v2.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/project-context.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/sprint-status.yaml"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/21-1-k6-baseline-closure.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/21-2-postgresql-ha-migration-managed-rds-multi-az-or-equivalent.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/epic-18-retro-2026-05-04.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/epic-19-retro-2026-05-04.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/epic-20-retro-2026-05-04.md"
focus: "IR-v6 — operator-requested implementation-readiness check at the Epic 21 mid-flight boundary. Closes the pre-epic-IR gate that IR-v5 named non-negotiable but that was bypassed for E19, E20, and E21-entry (CR-10). Scope: validate Epic 21 specification coherence with PRD §NFR + prd-amendment + architecture ADR-010 + the 6-story PE.01..PE.06 breakdown + Story 21-1 (done) and 21-2 (review) state + readiness for 21-3..21-6 dispatch. Reconciles state since IR-v5 (2026-05-04 03:16) and v21 sprint-change-proposal (2026-05-04 22:19)."
date: 2026-05-04
assessor: "PM (BMAD autopilot, bmad-check-implementation-readiness)"
supersedes: "implementation-readiness-report-2026-05-04-v2.md (IR-v5)"
gate: "Epic 21 / Story 21-3..21-6 dispatch"
---

# Implementation Readiness Assessment Report — IR-v6 (Epic 21 scope)

**Date:** 2026-05-04
**Project:** EU Solicit
**Mode:** BMAD autopilot (no operator prompts; no menus)
**Assessor role:** Product Manager — requirements traceability + planning-hygiene critique
**Trigger:** v21 sprint-change-proposal (2026-05-04 22:19) named `bmad-check-implementation-readiness` scoped to Epic 21 (IR-v6) as the **single highest-priority operative dispatch** — replacing IR-v5's §Recommended Next Steps #1 which was discharged via Story 18-1 Pass-2 Approve. Operator BMAD-stream rule: "Before starting any epic, run [IR] Implementation Readiness."
**Scope:** Epic 21 (Platform Reliability for 99.9% SLA) — 6 stories, 34 points, parallel to Epics 14–20 feature work; gates the public 99.9% SLA announcement.

---

## Executive Verdict

**Overall Readiness for Epic 21 dispatch: ⚠ PROCEED WITH NAMED CAVEATS.**

**HALT determination: NO HALT.** Epic 21 specification is coherent across all four canonical artefacts (epic file ↔ prd-amendment ↔ architecture ADR-010 ↔ ux-spec admin observability). Story 21-1 (k6 baseline closure) is `done` and produced the substrate (load-test-results.md §EXPLAIN ANALYZE + §Sizing Recommendations) that Stories 21-2..21-5 hard-depend on. Story 21-2 (PostgreSQL HA + M_PE02_opportunities_tsv_gin_index) is `review` awaiting bmad-code-review Pass-2 Approve. Stories 21-3..21-6 remain `backlog` and should NOT be dispatched until 21-2 closes.

**The operative blocker for Epic 21 progression is Story 21-2 Pass-2 Approve.** This is the AP17-C1 two-gate-close 10th potential recurrence. If it lands clean, the 5-in-a-row Approve streak (S19-0/19-1/19-2/20-0/21-1) extends to 6.

The five carry-forwards that were re-elevated by IR-v5, sprint-change-proposal v20/v21, and the three new retrospectives (E18/E19/E20) remain. Two are operative pre-conditions to the public SLA promise that Epic 21 backs:

1. **inj-01 Dependabot configuration** — `ready-for-dev`, 17 epic boundaries late. Per epic-20-retro AP20-C2 line 337, named in the **Critical-prep-before-E21** list. Cumulative unscanned dependency count grew across E18/19/20.
2. **CR-4 PRD source unification** — `PRD.md` still carries NFR-14 = 99.5% uptime; the 99.9% target lives only in `prd-amendment-2026-04-25.md`. Architecture ADR-010 + Epic 21 + the public SLA promise all read from the amendment. Material risk if PE.05 SLO dashboards or PE.06 runbooks are produced against the wrong NFR-14 number.

| Dimension | Verdict | Delta vs IR-v5 |
|---|---|---|
| Epic 21 spec coherence (epic file ↔ ADR-010 ↔ amendment) | ✅ Strong | E21 file last touched 2026-05-04 21:39 — Amendment 2026-05-04 (lines 148–173) was added inline to elevate `M_PE02_opportunities_tsv_gin_index` to a hard prerequisite; PE.02 implementation block at line 76 is in-place. |
| PRD coverage of Epic 21 (NFR-14 = 99.5% vs 99.9%) | 🛑 Split-source | `PRD.md` mod 2026-04-27 21:57 — still has NFR-14 = 99.5%. Amendment is the sole source of 99.9%. **CR-4 aged 9 → 10 working days.** |
| Architecture coverage of Epic 21 (ADR-010 + §6.5 + §6.6) | ✅ Strong | `architecture.md` mod 2026-05-04 21:39 — ADR-010 line 757 explicit; §6.5 lines 628–630 specify HPA + PDB minAvailable:1 + RDS Multi-AZ + Redis Sentinel pinned to "Epic 21". |
| UX coverage of Epic 21 (admin observability + status page) | ⚠ Acceptable | Epic 21 has zero customer-facing UX surface (admin/SRE-internal). ux-spec §3 line 150 names "page on-call" admin incident flow as goal; no public status-page UX in spec yet. Non-blocking for PE.01..PE.04 dev; **may surface as gap before public SLA announcement** (status-page is a customer-trust artefact). |
| Story 21-1 (k6 baseline closure) | ✅ Done | Pass-2 Approve verdict on disk; load-test-results.md populated with FTS EXPLAIN ANALYZE evidence (Seq Scan + 289 ms at 10K rows + extrapolation showing NFR-13 fail at 1M rows) → triggers PE.02 amendment. |
| Story 21-2 (PostgreSQL HA + tsv GIN) | ⚠ In review | Status `review`; awaiting bmad-code-review Pass-2 Approve. AP17-C1 10th recurrence potential. **Operative dispatch #1.** |
| Stories 21-3 / 21-4 / 21-5 / 21-6 | 🛑 Backlog (correct state) | Should not be dispatched until 21-2 Approve lands; PE.03–PE.05 read PE.02 outputs (HA primitives + ESO patterns + EXPLAIN evidence). |
| inj-01 Dependabot (CR-6 / AP20-C2 / Critical-prep-before-E21) | 🛑 CRITICAL — `ready-for-dev`, 17 epic boundaries late | Identified 13th-deferral in IR-v4; named in epic-20-retro Critical-prep-before-E21 list (`Dependabot merged`); not dispatched. |
| inj-02 k6 baseline supersession | ⚠ Audit-trail gap | sprint-status.yaml line 297 still shows `inj-02-k6-performance-baseline: ready-for-dev`. Story 21-1 AC-10 specifies "ready-for-dev → superseded" reconciliation at story-done time; **not landed in the same commit as 21-1: done**. Non-blocking on 21-2 dispatch but historic record diverges from epic AC-10 promise. |
| CR-9 retroactive 5 re-passes (17-0..17-3 + 18-0) | 🛑 Unresolved | Forward streak intact (S18-1 → S21-1 = 6 stories two-gate); retroactive 5 still owed for audit trail. |
| CR-1 + CR-2 hygiene (89 epic files, `epics.md` 9-epic legacy) | 🛑 Unresolved | `epics.md` mod 2026-04-30; `_archive/` dirs absent; `epics/` directory now 89 files. Aged 10 working days. |
| CR-7 UX gap for Epic 18 surfaces | ⚠ Operationally moot | E18 closed without explicit Sally pre-pass; surface debt persists only for AP18-C1 18-3-trust-center-hardening if injected. |
| CR-10 pre-epic IR gate | ⚠ Closing for E21 | This report (IR-v6) closes the gate for Epic 21 going-forward. Retroactive E19/E20 violation is now historic record (those epics already closed via retros 2026-05-04). |
| CR-11 AP20-C2 k6 absence (HARD BLOCK on Story 21-1) | ✅ DISCHARGED | Story 21-1 done with populated load-test-results.md per AC-7 grep gate. |

**Net change since IR-v5 (~19 hours, in line with v21 SCP timing): three epic closures (E18, E19, E20), one epic kickoff (E21), one critical path story closed (21-1), one second story dev-passed (21-2 → review), six new anti-patterns codified across three retros, two CRs added (CR-10, CR-11) one of which now discharged.**

The single highest-priority operative item replaces the v21 SCP §Operative routing #1: **dispatch `bmad-code-review` (Pass-2) for Story 21-2** to clear the AP17-C1 10th-recurrence gate and unlock 21-3..21-6 fan-out.

---

## Step 1 — Document Discovery

### Canonical documents (operative source of truth) — verified 2026-05-04

| Document | Path | Last modified | Status |
|---|---|---|---|
| PRD (base) | `planning-artifacts/PRD.md` | 2026-04-27 21:57 | Operative; **NFR-14 still = 99.5%** (CR-4) |
| PRD amendment | `planning-artifacts/prd-amendment-2026-04-25.md` | 2026-04-25 16:40 | Operative; **sole source of NFR-14 99.9%, FR8–FR11, Trust Center** |
| Architecture | `planning-artifacts/architecture.md` | 2026-05-04 21:39 | Operative; ADR-010 §757–762; §6.5 lines 628–630 reference Epic 21 explicitly |
| UX spec | `planning-artifacts/ux-spec.md` | 2026-05-04 02:06 | Operative; admin/incident flow §3 line 150 only — no public status-page UX |
| Epics roadmap (legacy) | `planning-artifacts/epics.md` | 2026-04-30 12:16 | **Drifted** — 9-epic legacy view; E10–E21 invisible (CR-2) |
| Epic 21 spec | `planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` | 2026-05-04 21:39 | Operative; Amendment 2026-05-04 lines 148–173 in-place; PE.02 implementation block line 76 |
| Sprint status | `implementation-artifacts/sprint-status.yaml` | 2026-05-04 22:03 | Operative; epic-21 in-progress; 21-1 done; 21-2 review; 21-3..21-6 backlog |
| project-context | `planning-artifacts/project-context.md` | 2026-05-04 16:45 | Operative |
| Latest sprint-change-proposal | `planning-artifacts/sprint-change-proposal-2026-05-04-v3.md` | 2026-05-04 22:19 | v21 of bmad-correct-course; routes to this IR-v6 |
| IR-v5 (predecessor) | `planning-artifacts/implementation-readiness-report-2026-05-04-v2.md` | 2026-05-04 03:16 | Operative until this IR-v6 supersedes |
| Story 21-1 | `implementation-artifacts/21-1-k6-baseline-closure.md` | 2026-05-04 | Status `done` (Pass-2 Approve) |
| Story 21-2 | `implementation-artifacts/21-2-postgresql-ha-migration-managed-rds-multi-az-or-equivalent.md` | 2026-05-04 | Status `review` (dev-pass complete; Pass-2 Approve pending) |
| Three new retros | `implementation-artifacts/epic-{18,19,20}-retro-2026-05-04.md` | 2026-05-04 | All three completed; 13 net-new anti-pattern entries; epic-21 retro listed `optional` |

### Hygiene status (verified 2026-05-04)

```
$ ls planning-artifacts/_archive/        → does not exist
$ ls planning-artifacts/epics/_archive/  → does not exist
$ ls planning-artifacts/epics/ | wc -l   → 89 files (mix of legacy + canonical E**-*.md)
$ test -f .github/dependabot.yml         → does not exist (CR-6 / inj-01 unresolved)
```

**CR-1 + CR-2 + CR-6 (`inj-01`) remain unresolved on disk.** Aged 10 working days each. Recommendation: bulk-move ~68 stale `epic-N(N)-*.md` snapshots to `epics/_archive/` in a single hygiene pass (lossless; preserves history).

### Operative epic file set

The 21 canonical `E**-*.md` files (E01..E21) remain the operative set. Per IR-v5 §Duplicate / superseded epic-file warning, the ~68 legacy `epic-NN-*.md` files are non-operative. No file deletions are recommended — archival only.

---

## Step 2 — PRD Analysis (Epic 21 scope)

### Epic 21 PRD provenance

Epic 21 explicitly cites in §Source line 4: *"PRD v1.1 §7 NFRs; architecture-evaluation §2 Change-5 (architect surfaced this as a separate workstream)"*. The 99.9% target is the operative NFR claim for Epic 21.

### CR-4 — PRD source split (UNRESOLVED, aged 10 working days)

| Source | Document | Statement |
|---|---|---|
| Base PRD | `PRD.md` line 60 | *"Platform availability of >= 99.5% for all services."* |
| Base PRD | `PRD.md` line 291 | *"NFR-14 (Uptime): The platform must achieve >= 99.5% uptime, measured monthly."* |
| Base PRD | `PRD.md` line 324 | *"the platform must maintain >= 99.5% availability"* |
| Amendment | `prd-amendment-2026-04-25.md` line 30 | *"v2.0 ... incorporates the 2026-04-25 PRD amendment (... 99.9% SLA, outcome telemetry)"* |
| Amendment | `prd-amendment-2026-04-25.md` line 48 | *"Tighten uptime to 99.9%"* |
| Architecture | `architecture.md` line 54 | *"99.5% MVP / 99.9% post-amendment uptime (NFR-14)"* (correct dual-anchor) |
| Architecture | ADR-010 line 757 | *"Boring infra for 99.9% SLA: Multi-AZ Postgres + Redis Sentinel, not Patroni"* |
| Epic 21 | E21 file line 3 | *"99.9% SLA externally publishable"* |
| Epic 21 | E21 file line 10 | *"99.9% = 43min downtime/month max"* |

**Operative target: 99.9%.** PRD.md is the only artefact still carrying 99.5% as a literal NFR-14 number. Material risk: when PE.05 (SLO dashboards) and PE.06 (on-call runbooks) ship, error-budget calculations and alert thresholds MUST anchor to 99.9% (43 min/month) not 99.5% (217 min/month). If any developer or QA reviewer reads `PRD.md` as the canonical NFR source, an entire month of error-budget math could be calibrated to the wrong number. **Recommended fix: PRD source unification before PE.05 dev dispatch.**

### Coverage of FR8–FR11 (PRD-amendment-only material)

These were already split-source in IR-v4 / IR-v5 (CR-4). Status unchanged at IR-v6:

| FR | Subject | Source | E** epics covering it |
|---|---|---|---|
| FR8 | Multi-Client Workspace | Amendment lines 71–86 | E14 (done) |
| FR9 | Outcome Telemetry | Amendment | E19 (done) |
| FR10 | Trust Center & Compliance | Amendment lines 173–179 | E18 (done — 18-0 + 18-1 + 18-2 all closed) |
| FR11 | CRM/Comms Integrations | Amendment lines 135–140 | E16 (done — Slack/Teams) + E17 (done — CRM) |

Coverage is complete in epic specs and three of four are now implemented. CR-4 is therefore a **PRD-document-hygiene** problem, not a feature-coverage problem. The operative material harm of CR-4 going forward is the 99.9% number for Epic 21 (above), not FR8–FR11.

### NFR coverage — Epic 21 maps

| NFR | Epic 21 story | Acceptance gate |
|---|---|---|
| NFR-1 (p95 < 200ms REST) | PE.01 (k6 baseline — done) | k6 thresholds set in 21-1 §AC-2; baseline numbers in load-test-results.md |
| NFR-2 (TTFB < 500ms SSE) | PE.01 (done) | Custom `sse_ttfb_ms` Trend metric per 21-1 §AC-3 |
| NFR-3 (Lighthouse ≥ 90) | NOT in scope of Epic 21 (frontend-shell concern; covered by E03) | — |
| NFR-4 (100 concurrent active users) | PE.01 (done) | k6 concurrent VU shapes |
| NFR-12 (read replicas) | PE.02 (review) | architecture.md §6.5 line 629 — "1 read replica for analytics workloads" |
| NFR-13 (10K companies / 1M opportunities at <20% degradation) | PE.02 (review) — `M_PE02_opportunities_tsv_gin_index` migration | Bitmap Index Scan EXPLAIN ANALYZE evidence + extrapolation calc |
| NFR-14 (uptime — 99.9% per amendment) | PE.02 + PE.03 + PE.04 + PE.05 + PE.06 | E21 §AC-1..AC-9; gates public SLA announcement per E21 line 20 |
| NFR-17 (DR within 4 hours) | PE.02 (review) — Multi-AZ failover | 35d backup retention; failover < 30s per 21-2 AC-6 |
| NFR-23 (structured logs / Prometheus / Grafana) | PE.05 (backlog) | Re-homes Epic 13 carry-forward |

**No PRD/NFR coverage gaps for Epic 21.** All NFRs that Epic 21 claims to back are mapped to a specific PE story with measurable AC.

---

## Step 3 — Epic Coverage Validation (Epic 21 internal)

### Story-by-story coherence

| Story | Pts | Type | State | AC count | Hard inputs satisfied | Hard outputs gating downstream |
|---|---|---|---|---|---|---|
| 21-1 k6-baseline-closure (PE.01) | 5 | platform-engineering | **done** | 10 | inj-02 carry-forward; existing k6-perf-core-flows.js + k6-agent-endpoints.js; staging-seed-perf-baseline.py | load-test-results.md §EXPLAIN ANALYZE + §SSE Methodology + §Queue Depth Timeline + §Sizing Recommendations for PE.02–PE.04 (drives 21-2/3/4 sizing); inj-02 supersession (AC-10 — see audit-trail finding below) |
| 21-2 postgresql-ha-migration (PE.02) | 8 | platform-engineering | **review** | 7 | E21 Amendment 2026-05-04 lines 148–173; 21-1 EXPLAIN ANALYZE evidence; per-service DB roles in 01-init-schemas-and-roles.sql | Bitmap Index Scan regression evidence (closes AC-2.4 deviation from 21-1); ESO ExternalSecret pattern; pe-02-cutover-runbook.md (PE.06 references); failover-drill-results (PE.06 references) |
| 21-3 redis-ha-migration (PE.03) | 5 | platform-engineering | backlog | TBD | PE.02 ESO pattern; 21-1 Redis 10K-INCR baseline | Sentinel-aware redis-py config; failover < 10s; consumer-group offset retention (PE.06 references) |
| 21-4 poddisruptionbudgets (PE.04) | 5 | platform-engineering | backlog | TBD | PE.01 baseline numbers (HPA sizing); PE.02 + PE.03 cutover artefacts | PDB CRD per-service; min-replica counts (client-api=3, others=2); helm-chart-lint CI step |
| 21-5 slo-dashboards (PE.05) | 8 | platform-engineering | backlog | TBD | PE.01 + PE.02 + PE.03 + PE.04 (need real prod traffic shape); Epic 13 Prometheus /metrics carry-forward | Grafana dashboards; Alertmanager rules; error-budget burn-rate alerts (PE.06 references) |
| 21-6 on-call-runbooks (PE.06) | 3 | process | backlog | TBD | PE.05 dashboards; PE.02 cutover-runbook + failover-drill; PE.03 Sentinel runbook | First simulated incident post-mortemed; 10 runbooks; on-call schedule active ≥ 2 weeks before public SLA |

**34 points total across 6 stories** — matches E21 line 3 spec.

### Sequencing — verify the dependency graph is sound

```
21-1 (PE.01)  ──────────────────────────────────────────┐
   │                                                    │
   └──► 21-2 (PE.02) ──┬──► 21-4 (PE.04) ──┐           │
   │                   │                    │           │
   └──► 21-3 (PE.03) ──┘                    ├──► 21-5 ─┴──► 21-6
                                            │      (PE.05)   (PE.06)
                                            │
   PE.01 baseline numbers also feed PE.04 directly (HPA sizing)
   PE.01 baseline numbers also feed PE.05 directly (alert thresholds)
```

- **21-1 (done) is critical-path and produced.** ✅
- **21-2 (review) is the operative blocker.** PE.04 cannot dispatch without 21-2 ESO pattern (env-var contract for new connection strings). PE.05 dashboards reference PE.02 EXPLAIN ANALYZE evidence pattern as canonical regression test.
- **21-3 (backlog) is parallelisable with 21-2.** Sentinel migration touches different infra surface (ElastiCache vs RDS) and uses 21-2's ESO pattern as input. Can run in parallel after 21-2 Approve lands.
- **21-4 (backlog) requires 21-2 + 21-3 cutovers complete** so PDB targets the new HA-aware deployments.
- **21-5 (backlog) requires 21-1+21-2+21-3+21-4** for real metrics and threshold calibration.
- **21-6 (backlog) requires 21-5** so runbooks reference live dashboards.

**Sequencing is sound.** No dispatch-order anti-patterns observed. Epic 21 §AC-7 ("Public SLA announcement gated: cannot publish until PE.01 + PE.02 + PE.03 + PE.04 ship") correctly identifies the four hard prerequisites for the public SLA — PE.05 + PE.06 are necessary for ops sustainability but not on the announcement gate.

### Audit-trail finding — inj-02 supersession not landed

Story 21-1's AC-10 (per sprint-status line 339) specifies: *"sprint-status carry-forward reconciliation at done-time (inj-02 ready-for-dev → superseded with comment)"* and: *"inj-02 will transition ready-for-dev → superseded ATOMICALLY with 21-1 backlog → done in the AC-10 reconciliation patch"*.

**Verified state:** sprint-status.yaml line 297 reads `inj-02-k6-performance-baseline: ready-for-dev`. The atomic supersession patch was NOT landed when 21-1 transitioned to `done`.

**Severity: LOW (audit-trail / hygiene).** Non-blocking on 21-2 dispatch. The 6-epic-carry audit trail is preserved on disk (per AC-10's "NOT deleted" clause), but the status field still reads `ready-for-dev` instead of `superseded`. Recommended fix: surgical 1-line edit during the next sprint-status touch (e.g., when 21-2 transitions to `done`, fold inj-02 supersession into the same atomic patch). **Add to AP17-C1 / AP18-C2 streak protection list:** future story-done patches MUST execute their epic AC reconciliation in the same commit as the story-done flip.

### Coverage of Epic 13 carry-forwards in Epic 21 scope

| Carry-forward | Status at IR-v6 | Re-home | Rationale |
|---|---|---|---|
| inj-01 dependabot | `ready-for-dev` (17 boundaries late) | NOT in Epic 21 scope per E21 line 8 ("Dependabot remains in Epic 13's set") | Should land before E21 closes for security hygiene; named in epic-20-retro Critical-prep-before-E21 |
| inj-02 k6 baseline | `ready-for-dev` (audit-trail bug — see above) | **PE.01 (21-1) — done** | Supersession patch not landed |
| inj-03 TEA review backlog | `ready-for-dev` | NOT in Epic 21 scope | TEA gate is process-layer, not infra |
| drift-recovery-story | `ready-for-dev` | NOT in Epic 21 scope | Epic 13 closure carry — separate dispatch |
| dw-01..03 | `ready-for-dev` | NOT in Epic 21 scope | Epic 7 + 9 deviation work |

**Conclusion:** Epic 21 correctly re-homes only PE.01 (k6 baseline) and PE.05 (Prometheus /metrics). The other six Epic-13 carry-forwards remain outside Epic 21 scope per the epic spec line 8. This is the correct scope decision. **inj-01 (Dependabot) is the only one that operationally MUST land before E21 closes** (security hygiene; epic-20-retro Critical-prep-before-E21).

---

## Step 4 — UX Alignment

### Epic 21 UX surface area

Epic 21 has **zero customer-facing UX surface for PE.01..PE.04** (infra-only stories). PE.05 produces Grafana dashboards (admin/SRE-internal), PE.06 produces runbooks (process). The only customer-visible artefact Epic 21 backs is the **public SLA announcement itself** — which is gated by E21 §AC-7 and is a marketing/legal/trust-center artefact, not an in-app screen.

### ux-spec coverage check

| ux-spec touchpoint | Epic 21 relation | Status |
|---|---|---|
| §3 line 150 admin "page on-call" incident flow | PE.06 on-call rotation feeds this admin surface | ⚠ admin UX exists abstractly; specific page-on-call IA / wireframe absent. **Non-blocking for PE.06 dev** (process-layer story); may surface as gap when PE.06 needs to wire a real PagerDuty/Opsgenie webhook into an admin "Incidents" surface. |
| §3 line 303 admin "Files an internal incident" → `Admin → Incidents → New` | PE.06 incident-management process | ⚠ admin path exists in IA; no detail wireframe. **Non-blocking; deferrable to PE.06 dev**. |
| §7.9 audit and trust signals | Public SLA announcement (post-E21 milestone) | ⚠ E18 Trust Center surfaces are present (ux-spec §7.9). Public status-page UX is NOT in spec. **Will become a gap when public SLA announcement ships** (post-PE.04). |

### Public status-page UX — surfaced gap (NEW finding)

Epic 21 backs a public 99.9% SLA promise. Industry norm for any vendor publishing a 99.9% SLA is a public status-page (statuspage.io, Atlassian Statuspage, or self-hosted Cachet) showing real-time uptime per service component. **The current ux-spec.md does NOT define a public status-page surface.** This is a UX coverage gap that does NOT block PE.01..PE.06 dev but DOES block the public-SLA-announcement milestone that E21 §AC-7 names.

**Recommendation:** Treat as **deferrable backlog injection**: when PE.05 (SLO dashboards) lands, file an `inj-XX-public-status-page-ux` story (or fold into PE.06 scope as an additional AC). Effort estimate: 1–2 pts (UX spec append + UI wireframe + integration with PE.05 dashboards). **Tag: CR-12 (NEW).**

---

## Step 5 — Epic Quality Review

### Epic 21 spec quality

| Quality dimension | Status | Evidence |
|---|---|---|
| Goal statement | ✅ Strong | Line 7 — clear product-vision link ("99.9% uptime SLA promised in PRD v1.1 §7"); explicit blocker statement ("Without this work, the SLA cannot credibly be published") |
| Acceptance criteria | ✅ Strong | 9 AC bullets at lines 14–22; AC-7 names exact gate condition ("cannot publish until PE.01 + PE.02 + PE.03 + PE.04 ship") |
| Story breakdown | ✅ Strong | 6 stories with explicit point sizing (5/8/5/5/8/3 = 34 pts); type tag per story (platform-engineering / process); critical-path call-out at PE.01 |
| Per-story scope detail | ✅ Strong | Each story has Scope + Tests blocks; PE.02 has Implementation block + Amendment block; PE.05 explicitly re-homes Epic-13 Prometheus carry-forward |
| Dependency call-outs | ✅ Strong | PE.01 → PE.02/3/4 sizing input; PE.02 cutover-runbook → PE.06 reference; PE.05 → PE.06 reference |
| Risk surface | ⚠ Partial | E21 §AC-8 documents KraftData circuit-breaker exemption (good); §AC-9 documents two-layer resilience pattern (good). **Missing: explicit risk that the 99.9% target itself may be unachievable on first 30 days post-cutover** (industry baseline: managed PG Multi-AZ + ElastiCache HA typically deliver 99.95–99.99% but the platform-as-a-whole is bottlenecked by the lowest-availability dependency, often a single-region dependency). Recommend documenting this as Known Deviation when PE.05 monitors first-30-day error budget. |
| Test design | ⚠ No formal test-design-epic-21.md | Per Story 21-1 §D-7 and Story 21-2 §D-7 pattern: no `test_artifacts/test-design-epic-21.md` exists; consistent with E19/E20 fallback. Acceptable for non-functional baseline epic whose quality gate is a populated evidence file (load-test-results.md) and a captured failover-drill (pe-02-cutover-runbook.md). |
| Anti-pattern fences in story files | ✅ Strong (Story 21-1 + 21-2 each have 8-row fences) | Per-story specific anti-patterns documented; AP17-C1 + AP18-C2 streak-protection clauses in both stories |

### Stories 21-1 and 21-2 quality (the two with story files extant)

**Story 21-1** (10 ACs, 12-task breakdown, 8-row anti-pattern fence, 7 Known Deviations §6 pre-recorded): excellent depth. Closed via Pass-2 Approve.

**Story 21-2** (7 ACs, 9-task breakdown, 8-row anti-pattern fence, 2 Known Deviations §6 pre-recorded — D-1 production cutover deferred to operator-action; D-2 staging rehearsal becomes "documented dry-run"): excellent depth. Status `review` awaiting Pass-2 Approve. **No quality concerns observed.**

### Story files for 21-3..21-6: NOT YET CREATED

This is the correct state per Operator BMAD-stream rule "Before each story, ALWAYS run [VS] Validate Story" and the established pattern (story files are created by `bmad-create-story` in the same atomic patch as the story's backlog → ready-for-dev status flip). PE.03..PE.06 story files will be authored when their dispatch becomes operative. **No deficiency.**

### Test-design provenance

No `test_artifacts/test-design-epic-21.md` exists. Per the consistent pattern across E19/E20/Story-21-1/Story-21-2 D-7 entries, this is the established fallback for non-functional baseline epics: implicit test design is the populated evidence-file regression-test pattern (load-test-results.md §EXPLAIN ANALYZE comparison; pe-02-cutover-runbook.md §Failover Drill Results comparison). **Acceptable; non-blocking for E21 dispatch.**

---

## Step 6 — Final Assessment

### Cross-Reference Matrix (CR table refreshed for IR-v6)

| CR | Severity | Title | Aged | Status @ IR-v6 |
|---|---|---|---|---|
| CR-1 | 🛑 HIGH | Planning-artifact hygiene (~68 stale epic-NN-*.md files; `_archive/` dirs absent) | 10 wd | Unresolved on disk. **Recommended hygiene pass before Epic 22.** |
| CR-2 | 🛑 HIGH | Planning-view drift (`epics.md` 9-epic legacy view) | 10 wd | Unresolved. `epics.md` mod 2026-04-30. |
| CR-3 | ✅ CLOSED | Epic 16 status integrity | — | — |
| CR-4 | 🛑 HIGH | PRD source unification (NFR-14 99.5% vs 99.9% in amendment) | 10 wd | Unresolved. **Material risk: PE.05 SLO threshold calibration must anchor to 99.9% from amendment, not 99.5% from PRD.md.** |
| CR-5 | ⚠ LOW | UX gap for E17 surfaces | — | Backlog tech debt; E17 closed without inline mitigation. |
| CR-6 | 🛑 CRITICAL | Epic 13 carry-forward `inj-01` Dependabot | 17 epic boundaries late | Unresolved. **Named in epic-20-retro Critical-prep-before-E21 list. Cumulative unscanned dependency count grew through E18/19/20.** |
| CR-7 | ⚠ MEDIUM | UX gap for E18 surfaces | — | Operationally moot post-E18-close; surface debt persists for AP18-C1 18-3-hardening if injected. |
| CR-8 | ✅ DISCHARGED | E18 design questions (R-018-2/3/5) | — | Resolved inline. |
| CR-9 | ⚠ MEDIUM | AP17-C1 quintuple-fire — retroactive 5 re-passes (17-0..17-3 + 18-0) | — | Forward streak intact (S18-1 → S21-1 = 6-story streak). Retroactive 5 re-passes still owed. |
| CR-10 | ⚠ CLOSING | Pre-epic IR gate violated (E19, E20, E21-entry) | — | **Closing for E21 going-forward via this IR-v6.** Retroactive E19/E20 violation is now historic record. |
| CR-11 | ✅ DISCHARGED | AP20-C2 k6 absence — HARD BLOCK on Story 21-1 | — | Discharged via Story 21-1 Pass-2 Approve + populated load-test-results.md. |
| **CR-12 (NEW)** | ⚠ MEDIUM | **Public status-page UX absent from ux-spec** | NEW @ IR-v6 | Non-blocking for PE.01..PE.06 dev; **blocks public-SLA-announcement milestone** (post-PE.04). Recommend deferrable injection candidate (1–2 pts) folded into PE.06 scope or filed as `inj-XX-public-status-page-ux`. |
| **CR-13 (NEW)** | ⚠ LOW | **Story 21-1 AC-10 reconciliation incomplete: `inj-02` still `ready-for-dev`** | NEW @ IR-v6 | Audit-trail / hygiene. Surgical 1-line fix during next sprint-status touch (atomic with 21-2 done). |

### Recommended Next Steps (operative dispatch order)

#### Immediate (next 2–4 hours)

1. **🛑 CRITICAL — Dispatch `bmad-code-review` (Pass-2) for Story 21-2.** AP17-C1 10th-recurrence gate. Approve verdict required for `done` transition. Protects S19-0/19-1/19-2/20-0/21-1 5-story streak; if successful, extends to 6-story streak. **Effort: ~30–45 min.** Gates 21-3..21-6 dispatch.

2. **🛑 CRITICAL — Dispatch `inj-01-dependabot-configuration` in parallel.** Single file (`.github/dependabot.yml`); <30 min. Per epic-20-retro Critical-prep-before-E21 list. Can run in parallel with #1 (no conflict). After landing, retroactively scan dep additions from E18/E19/E20 stories for CVE findings.

#### After Story 21-2 closes Approved

3. **`[VS] Validate Story` for Story 21-3 (Redis HA).** Non-negotiable per Operator workflow guidance. Reads PE.02 ESO pattern as input.

4. **`bmad-create-story` for Story 21-3.** AP18-C2 atomic patch (story file Status + sprint-status development_status flip in same commit). Include Story 21-1 AC-10 reconciliation patch in same commit (closes CR-13: `inj-02-k6-performance-baseline: ready-for-dev → superseded`).

5. **`bmad-dev-story` → `bmad-code-review` for Story 21-3.** AP17-C1 11th-recurrence gate. Two-gate enforcement.

6. **`[SR] Story Review` for Story 21-3.** Non-negotiable per Operator workflow guidance for multi-story epics.

#### Parallelisation opportunity

7. **Story 21-4 (PDB + min-replica) can dispatch in parallel with 21-3 dev** if 21-2 ESO pattern + 21-1 baseline numbers are sufficient inputs. Verify: PE.04 scope reads PE.02 cutover artefact OR only PE.02 + PE.03 deployment manifests. If only the latter, **21-4 is gated on 21-3 done**; if the former, **21-4 can dispatch parallel to 21-3 dev**. Recommend operator clarification at story-create time.

#### After 21-3 + 21-4 close

8. **`[VS] Validate Story` → `bmad-create-story` → `bmad-dev-story` → `bmad-code-review` → `[SR] Story Review` for Story 21-5 (SLO dashboards).** Highest-effort story (8 pts); reads 21-1+21-2+21-3+21-4 outputs. **Critical: alert thresholds and error-budget calculations MUST anchor to 99.9% NFR-14 from prd-amendment, NOT 99.5% from PRD.md** (CR-4).

#### After 21-5 closes

9. **`[VS] Validate Story` → ... → `[SR] Story Review` for Story 21-6 (on-call + runbooks).** Reads 21-2 cutover-runbook + 21-3 Sentinel runbook + 21-5 dashboards. Folds `inj-XX-public-status-page-ux` (CR-12) here OR files separately at operator discretion.

#### After Story 21-6 closes Approved

10. **`[ER] Epic Review` for Epic 21.** Mandatory per Operator BMAD-stream guidance ("For epics with complex or interdependent stories, run [ER] Epic Review after all stories are complete"). Epic 21 is the most-interdependent epic in the project (PE.01→PE.02/3/4 sizing chain; PE.04→PE.05 manifest chain; PE.05→PE.06 runbook chain). [ER] is non-negotiable.

11. **`bmad-retrospective` for Epic 21** (currently `optional`; recommended given Epic 21 is the platform's first multi-quarter-impact reliability workstream).

12. **`[PR] Post-Review` for Epic 21** before public SLA announcement. Verifies: (a) 99.9% NFR-14 number used consistently across SLO dashboards + runbooks + alert thresholds (close CR-4); (b) public status-page UX delivered (close CR-12); (c) 30-day live error-budget burn-rate observed under target before announcement.

#### Carry-forward debt — should land BEFORE Epic 22 (NOT blocking E21)

13. **CR-1 + CR-2 + CR-4 batched hygiene pass.** ~30–60 min operator-level decision. **Recommend landing during Epic 21 mid-flight downtime** (between 21-2 Approve and 21-3 dispatch is a natural breakpoint). Prevents further epic-state-of-truth drift.

14. **Retroactive `bmad-code-review` re-passes for 17-0 / 17-1 / 17-2 / 17-3 / 18-0 (CR-9).** Audit-trail liability before any future GA-gate or external compliance audit. Sequencing: chronological. Effort: 30–45 min × 5 = ~3 hours. **Recommend before Epic 22 kickoff.**

15. **AP18-C1 18-3-trust-center-hardening injection.** Open backlog candidate. Operationally non-urgent (E19/E20/E21 do not surface trust-center pages). Operator-discretion injection.

#### Process-layer (orchestrator-dispatcher; out-of-scope for this skill)

16. **Orchestrator dispatcher 24h-suppression-window for autonomous-drift-detection trigger** when a `sprint-change-proposal-*` OR `implementation-readiness-report-*` exists in the last 24h. **22nd-fire pattern not yet observed but likely after IR-v6 lands** based on v18→v19→v20→v21 cadence. Mitigation responsibility: operator routing-layer.

---

## Verdict for Epic 21

**Specification readiness: ✅ READY FOR DISPATCH (21-2 Pass-2 Approve, then 21-3..21-6).**

Epic 21 is the cleanest-specified epic in the project's Reliability workstream. Coherence between epic file ↔ ADR-010 ↔ amendment ↔ story files (21-1, 21-2) is strong. Story 21-1 closed the highest-risk dependency (k6 baseline absence — 6-epic carry-forward; AP20-C2 hard-block) and produced the substrate that Stories 21-2..21-5 hard-depend on. Story 21-2 is one Pass-2 Approve away from unblocking the rest of the epic.

**No HALT.** The single highest-priority operative dispatch is `bmad-code-review` for Story 21-2 (Pass-2). The single parallel dispatch is `inj-01-dependabot-configuration` (Critical-prep-before-E21 from epic-20-retro).

**Two NEW carry-forwards to track for Epic 22 / public-SLA-announcement readiness:**
- CR-12 (public status-page UX)
- CR-13 (Story 21-1 AC-10 reconciliation — `inj-02` supersession not landed)

**Five existing carry-forwards still operative going into Epic 21:**
- CR-1 (~68 stale epic files)
- CR-2 (`epics.md` 9-epic legacy view)
- CR-4 (PRD NFR-14 99.5% vs 99.9% split — **material risk for PE.05 alert calibration**)
- CR-6 (`inj-01` Dependabot — 17 boundaries late, named in epic-20-retro Critical-prep-before-E21)
- CR-9 (5 retroactive Pass-2 re-passes owed)

**This IR-v6 closes CR-10 (pre-epic IR gate) for Epic 21 going-forward.** Retroactive CR-10 violation for E19 / E20 entry remains historic record (those epics are now closed via retros 2026-05-04).

---

## What this IR is *not*

- IR-v6 is **not** an artefact-edit proposal. Zero edits to PRD / architecture / epics / ux-spec / story files / sprint-status.yaml.
- IR-v6 is **not** a sprint-status edit. Per project memory rule (sprint-status.yaml is orchestrator-managed — surgical edits only). The CR-13 audit-trail correction is a recommendation for the next operator-driven sprint-status touch, not a direct edit by this IR.
- IR-v6 is **not** a re-press of v21 SCP's open recommendations. v21 §Operative routing #1 (dispatch this IR-v6) is now discharged. v21 §Operative routing #2 (`inj-01`) remains operative and is re-asserted as IR-v6 §Recommended #2.
- IR-v6 is **not** a supersession of v21 SCP. v21 remains on the audit trail as the operative reference for the historical state at 2026-05-04 22:19. IR-v6 is the operative reference for state from 2026-05-04 ~22:30 forward, until IR-v7 supersedes.

---

## Workflow execution log entry

```yaml
date: 2026-05-04
workflow: bmad-check-implementation-readiness
mode: BMAD autopilot (no operator prompts)
trigger: Operator-requested IR scoped to Epic 21 (per v21 SCP §Operative routing #1)
input_artifacts:
  - planning-artifacts/PRD.md (mod 2026-04-27 21:57)
  - planning-artifacts/prd-amendment-2026-04-25.md (mod 2026-04-25 16:40)
  - planning-artifacts/architecture.md (mod 2026-05-04 21:39)
  - planning-artifacts/ux-spec.md (mod 2026-05-04 02:06)
  - planning-artifacts/epics/E21-platform-reliability-99-9-sla.md (mod 2026-05-04 21:39)
  - planning-artifacts/sprint-change-proposal-2026-05-04-v3.md (v21, mod 2026-05-04 22:19)
  - planning-artifacts/implementation-readiness-report-2026-05-04-v2.md (IR-v5, mod 2026-05-04 03:16)
  - implementation-artifacts/sprint-status.yaml (mod 2026-05-04 22:03)
  - implementation-artifacts/21-1-k6-baseline-closure.md
  - implementation-artifacts/21-2-postgresql-ha-migration-managed-rds-multi-az-or-equivalent.md
  - implementation-artifacts/epic-{18,19,20}-retro-2026-05-04.md
output_artifacts:
  - planning-artifacts/implementation-readiness-report-2026-05-04-v3.md (this file, IR-v6)
edits_to_existing_artifacts: 0
status_transitions: 0
verdict: READY-WITH-CAVEATS (no HALT)
operative_routing: |
  1. bmad-code-review (Pass-2) for Story 21-2 — gates 21-3..21-6 dispatch
  2. inj-01-dependabot-configuration (parallel) — Critical-prep-before-E21
  3. After 21-2 Approve: [VS] → bmad-create-story for 21-3 (fold CR-13 inj-02 supersession in same commit)
  4. After 21-3 done: 21-4 (parallelise with 21-3 dev IF PE.04 inputs allow)
  5. After 21-3+21-4: 21-5 (alert thresholds MUST anchor to 99.9% — close CR-4)
  6. After 21-5: 21-6 (fold CR-12 status-page UX OR file as inj)
  7. After 21-6: [ER] → retro → [PR] before public SLA announcement
  8. Pre-Epic-22 hygiene: CR-1 + CR-2 + CR-4 batched pass
net_new_findings:
  - CR-12 (MEDIUM): public status-page UX absent from ux-spec; blocks public-SLA-announcement milestone post-PE.04
  - CR-13 (LOW): Story 21-1 AC-10 reconciliation incomplete — inj-02 still `ready-for-dev`; surgical fix during next sprint-status touch
discharged_findings:
  - CR-10 (closing for Epic 21 via this IR-v6)
  - CR-11 (Story 21-1 done discharges AP20-C2 k6 hard-block)
preserved_findings:
  - CR-1, CR-2, CR-4, CR-6, CR-9 (all unchanged in nature and severity)
scope_classification: Major (closes operative pre-epic-IR gate; surfaces 2 new CRs)
approval: auto-approved (BMAD autopilot)
handoff: bmad-code-review (Pass-2) for Story 21-2 [next-skill]; inj-01-dependabot-configuration [parallel]
```

---

*Generated by `bmad-check-implementation-readiness` skill in BMAD autopilot mode (no operator prompts) on 2026-05-04. This IR-v6 closes the pre-epic-IR gate (CR-10) for Epic 21 going-forward and routes the operative dispatch chain to Story 21-2 Pass-2 code review (the AP17-C1 10th-recurrence gate). Two new findings (CR-12 status-page UX, CR-13 inj-02 supersession audit-trail) are surfaced; both are deferrable. The five preserved long-standing CRs (CR-1, CR-2, CR-4, CR-6, CR-9) are unchanged from IR-v5; CR-4 carries the highest forward-going material risk (PE.05 alert calibration must anchor to 99.9% from prd-amendment).*
