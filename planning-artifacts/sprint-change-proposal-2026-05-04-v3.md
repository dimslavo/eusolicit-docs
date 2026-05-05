---
date: 2026-05-04
generator: bmad-correct-course (BMAD autopilot, no operator prompts)
trigger_brief: "Drift detected. Review requirements." (verbatim, identical to v2–v20; **21st consecutive fire**)
project: EU Solicit
scope_classification: "Process / sequencing — Minor (acknowledge state advance; defer routing to next-IR)"
supersedes: ""  # does NOT supersede v20; v20 + IR-v5 remain on the audit trail
defers_to: "next bmad-check-implementation-readiness invocation (IR-v6 owed pre-Epic-21; recommended scope = Epic 21)"
relationship_to_prior:
  - "v20 (2026-05-04 02:55) — post-IR-v4 / 20th-fire / no-new-action; recorded the 20th-fire-also-post-IR pattern"
  - "IR-v5 (2026-05-04 03:16) — reaffirmed IR-v4 in full; named pre-Epic-19 IR-v6 as 'non-negotiable per Operator BMAD-stream'"
  - "v21 fires ~19 hours after IR-v5; in that window Epics 18 / 19 / 20 all closed and Epic 21 entered in-progress without any IR-v6 dispatch"
no_artifact_edits: true   # no PRD / architecture / epics / ux-spec edits proposed
no_sprint_status_edits: true  # sprint-status.yaml is orchestrator-managed (project-context rule)
---

# Sprint Change Proposal — 2026-05-04 (v21, post-IR-v5 / 21st-fire / state-advance acknowledgement)

## TL;DR

**Substantive state advance since v20 / IR-v5 (~19 hours).** Three epic boundaries crossed (E18 → done, E19 → done, E20 → done) and Epic 21 entered `in-progress` with Story 21-1 (k6-baseline-closure) at `ready-for-dev`. **The pre-Epic-19 [IR] Implementation Readiness v6 dispatch — which IR-v5 explicitly named "non-negotiable per Operator BMAD-stream" — was not generated.** That gate was bypassed for E19, E20, AND E21.

**v21 proposes zero artefact edits.** No PRD, architecture, epics, ux-spec, or story-file changes. No sprint-status transitions (orchestrator-managed; surgical-edits-only per project memory rule). v21's single function is to record the state-advance and re-route the operative dispatch chain to a **fresh `bmad-check-implementation-readiness` (IR-v6) scoped to Epic 21**, replacing IR-v5's "Recommended Next Steps" list which is now substantially obsolete.

**Single highest-priority operative item (replaces IR-v5 #1):** dispatch `bmad-check-implementation-readiness` scoped to Epic 21 BEFORE Story 21-1 enters `bmad-dev-story`. Per epic-20 retro line 337, AP20-C2 (k6 absent for 15 consecutive epics) is now a **HARD BLOCK on E21 reliability work**, and Story 21-1 IS the closure of that carry-forward — so an IR-v6 verifying E21 spec coherence and 21-1's hard-dependencies is the gating precondition.

---

## Why this proposal exists

The `bmad-correct-course` workflow was dispatched for the 21st consecutive time with the verbatim brief *"Drift detected. Review requirements."* The orchestrator dispatcher continues not to consult prior `sprint-change-proposal-*.md` files OR `implementation-readiness-report-*.md` files as state before firing (the dispatcher-bias finding established in v19 §Operative observation, generalised in v20 §The 20th-fire meta-pattern, and now confirmed for the third successive instance).

Unlike v20 (which fired ≤45 minutes after IR-v4 closed and had zero substantive delta to record), **v21 fires ~19 hours after IR-v5 closed** and has substantial state delta. v21's job is therefore *not* a near-no-op — it is to acknowledge the state advance and re-route, since IR-v5's "Recommended Next Steps (1–8)" list is largely obsolete after three epic closures and Epic 21 entry.

---

## Substantive delta vs IR-v5 (2026-05-04 03:16) and v20 (2026-05-04 02:55)

### Epic state transitions in the v20 → v21 window (~19 hours)

| Epic | State at IR-v5 close (03:16) | State at v21 fire (~22:00) | Delta |
|---|---|---|---|
| 18 | `in-progress` (18-0 done w/o Approve, 18-1 review awaiting code-review pass-2, 18-2 backlog) | **`done`** (3/3 stories `done` via AP17-C1 two-gate; retrospective `done` 2026-05-04) | **MAJOR — full epic close.** AP17-C1 quintuple-fire **broken** by 18-1 → 18-2 two-gate-close streak. |
| 19 | `backlog` | **`done`** (3/3 stories `done`; retrospective `done` 2026-05-04) | **MAJOR — full epic open + close in single window.** 19-0 / 19-1 / 19-2 all closed via Pass-2 Approve. |
| 20 | `backlog` | **`done`** (1/1 story `done` — Story 20-0 NPS SDK + opt-in routing; retrospective `done`) | **MAJOR — full epic open + close.** Single-story epic; Story 20.2 (onboarding/CSM stall) was scope-deduplicated into Story 19-2 per epic-19 retro. |
| 21 | `backlog` | **`in-progress`** (Story 21-1 `ready-for-dev`; 5 stories still `backlog`) | **MAJOR — epic kickoff without pre-Epic-21 IR.** |

### Recommendation-vs-reality reconciliation (IR-v5 § Recommended Next Steps)

| IR-v5 #  | Recommendation | Status at v21 fire |
|---|---|---|
| 1 | Dispatch `bmad-code-review` for Story 18-1 (pass-2) | ✅ **DISCHARGED** — Round 3 review-fix landed; Pass-2 Approve verdict; 18-1 → `done`. |
| 2 | Sally pre-pass for E18 6 surfaces (CR-7) | ⚠ **BYPASSED** — 18-2 dispatched and closed without explicit Sally pre-pass; no append-only `ux-spec.md` supplements observed. **CR-7 unresolved at file level but operationally moot now that E18 is closed.** Surface debt persists for any future trust-center hardening (AP18-C1 18-3 candidate). |
| 3 | Dispatch `inj-01-dependabot-configuration` (Epic 13 carry-forward, mandatory pre-18-2) | 🛑 **NOT DISPATCHED** — `inj-01` still `ready-for-dev` (verified line in `development_status` block); E18 closed without it. **Now elevated by epic-20 retro (AP20-C2) to "Critical prep before E21: k6 run+committed, Dependabot merged".** |
| 4 | Sequential retroactive `bmad-code-review` re-passes for 17-0 → 17-1 → 17-2 → 17-3 → 18-0 (CR-9 / AP17-C1 quintuple-fire closure) | 🛑 **NOT DISPATCHED** — quintuple-fire audit-trail liability persists for those 5 stories. The pattern was BROKEN going forward (18-1, 18-2, 19-0, 19-1, 19-2, 20-0 = 6-story two-gate streak per epic-20 retro), but the original 5 retroactive re-passes were never landed. |
| 5 | CR-1 + CR-2 + CR-4 batched hygiene pass before Epic 19 kickoff | 🛑 **NOT DISPATCHED** — `epics.md` still 9-epic legacy view; `_archive/` dirs still absent; `prd-amendment-2026-04-25.md` still not merged into `PRD.md`. **Recommended-pre-Epic-19 gate violated.** |
| 6 | After 18-1 Approved → `[VS] Validate Story` for 18-2 → `bmad-create-story` → dev → review with two-gate enforcement | ✅ **DISCHARGED** — 18-2 closed via two-gate (Pass-2 Approve 2026-05-04). |
| 7 | Before Epic 18 closes: `[ER] Epic Review` then `bmad-retrospective` for E18 | ⚠ **PARTIALLY DISCHARGED** — `bmad-retrospective` ran (`epic-18-retro-2026-05-04.md` exists; 5 anti-patterns codified: AP18-C1 inject 18-3-hardening, AP18-C2 Status headers, AP18-C3 k6 gate 13th miss, AP18-C4 NFR regen, AP18-H4 TEA gate). `[ER] Epic Review` not visibly dispatched as a discrete artefact. |
| 8 | Pre-Epic-19: `[IR] Implementation Readiness` v6 (non-negotiable) | 🛑 **VIOLATED** — IR-v6 was not generated. Epic 19 entered, completed, AND was retrospected without IR-v6. Same gate violated for E20 (which also has no pre-epic IR) and is currently being violated for E21 entry. |

**Net IR-v5 recommendation discharge rate: 2/8 fully discharged (#1, #6); 2/8 partially discharged (#2 operationally moot, #7 partial); 4/8 not discharged (#3, #4, #5, #8).** The two non-blocking but recommended hygiene/process items (#5 + #8) are the most consequential carry-forward debt because they affect future epics (E21 and beyond), not just past ones.

### New anti-patterns codified across the three retrospectives

The three new retrospectives (E18, E19, E20) added 13 new anti-pattern entries to the project's audit trail:

- **Epic 18 retro (5 entries):** AP18-C1 (inject 18-3 trust-center-hardening for WeasyPrint D6/D7 — open backlog injection candidate), AP18-C2 (story file Status headers — propagated through E19, E20 as AP19-C1, AP20-C1), AP18-C3 (k6 gate 13th miss), AP18-C4 (NFR regen), AP18-H4 (TEA gate).
- **Epic 19 retro (6 entries):** AP19-C1 (Status headers 16th consecutive epic — sustained from AP18-C2), AP19-C2 (CurrentUser.workspace_id AttributeError — 3 trigger sites), AP19-C3 (concurrent.futures.wait+cancel not async-safe), AP19-C4 (aggregation reimplementation), AP19-H1 (test collection failure undetected), AP19-H2 (no ATDD checklist artifacts).
- **Epic 20 retro (8 entries):** AP20-C1 (Status headers 18th consecutive — AP18-C2 unresolved), **AP20-C2 (k6 absent 15th epic — E21 HARD BLOCK)**, AP20-C3 (TEA 15th epic), AP20-A1 (E2E all test.skip() at Approve), AP20-A2 (DEFERRABLE constraint IntegrityError post-route-return), AP20-A3 (frontend User type not extended when backend model extended), AP20-A4 (spec endpoint path diverged from implementation).

**The operative new HARD BLOCK is AP20-C2: k6 absent for 15 consecutive epics, surfacing as a hard precondition on Epic 21 reliability work** because Story 21-1 is the k6-baseline-closure story itself. Per the latest sprint-status entry (line 50): *"Critical prep before E21: k6 run+committed, Dependabot merged, E2E un-skip during [PR] Post-Review, dev-story template updated with 5 new anti-patterns, TEA embedded as AC gate."*

### `inj-01` (Dependabot) status — first concrete material harm escalates

IR-v4 / IR-v5 named `inj-01-dependabot-configuration` (Epic 13 carry-forward, 13th deferral) as the "first concrete material harm" marker because Story 18-1 shipped 3 unscanned WeasyPrint deps. v21 confirms:

- `.github/dependabot.yml` still verified absent on disk per IR-v5 §Hygiene status.
- `inj-01` still at `ready-for-dev` per `development_status` block.
- **Epic 18, 19, 20 all shipped to `done` with this gate unclosed.** Cumulative unscanned dependency count has grown beyond the original 3 (Stories 19-0/19-1/19-2/20-0 introduced additional package additions visible in retro entries — exact count requires audit but is non-zero).
- Per epic-20 retro (line 337): "Dependabot merged" is now in the **Critical prep before E21** list — i.e., elevated from "mandatory pre-18-2" (IR-v4) to "mandatory pre-21-1 dev". Same gate, four epic boundaries late.

### CR Summary Table — refreshed for v21

| CR | Severity at IR-v5 | Severity at v21 | Title | Status |
|---|---|---|---|---|
| CR-1 | 🛑 HIGH | 🛑 HIGH (aged 10 working days) | Planning-artifact hygiene (~68 stale epic files in `epics/`) | Unresolved on disk; `_archive/` dirs absent. |
| CR-2 | 🛑 HIGH | 🛑 HIGH (aged 10 working days) | Planning view drift (`epics.md` 9-epic legacy view) | Unresolved; `epics.md` last touched 2026-04-30. |
| CR-3 | ✅ CLOSED | ✅ CLOSED | Epic 16 status integrity | — |
| CR-4 | 🛑 HIGH | 🛑 HIGH (aged 10 working days) | PRD source unification (FR8–FR11 missing from `PRD.md`) | Unresolved; amendment still separate file. |
| CR-5 | ⚠ LOW | ⚠ LOW | UX gap for E17 surfaces | Backlog tech debt; E17 closed without inline mitigation. |
| CR-6 | 🛑 HIGH | 🛑 **CRITICAL** (aged 17 epic boundaries) | Epic 13 carry-forwards — `inj-01` Dependabot | **Escalated.** Now AP20-C2 + epic-20-retro Critical-prep-before-E21 list. |
| CR-7 | 🛑 CRITICAL | ⚠ MEDIUM (operationally moot post-E18-close) | UX gap for E18 surfaces | E18 closed without explicit Sally pre-pass; surface debt now applies only to AP18-C1 18-3-hardening if/when injected. |
| CR-8 | ⚠ MEDIUM | ✅ DISCHARGED (operationally) | E18 design questions (R-018-2/3/5) | Resolved inline through 18-1 / 18-2 dev passes. |
| CR-9 | 🛑 CRITICAL | ⚠ MEDIUM (forward-broken; retroactive 5 still owed) | AP17-C1 quintuple-fire | Going-forward: 6-story two-gate streak per epic-20 retro. Retroactive 5 re-passes (17-0→17-3 + 18-0) still unowed. |
| CR-10 | NEW | 🛑 **CRITICAL** | **Pre-epic IR-v6 gate violated for 3 consecutive epics (E19, E20, E21-entry)** | **NEW v21 finding.** Operator BMAD-stream "Before starting any epic, run [IR] Implementation Readiness" is documented in IR-v5 as non-negotiable. Three boundary crossings under that rule have occurred without compliance. |
| CR-11 | NEW | 🛑 **CRITICAL** | **AP20-C2 k6 absence — HARD BLOCK on Story 21-1** | **NEW v21 finding (carries epic-20 retro elevation).** Story 21-1 IS the k6-baseline-closure story; cannot start its dev without prior k6 baseline run + commit per epic-20 retro Critical-prep list. |

### Story 21-1 readiness

Per sprint-status line 338: *"21-1 (k6-baseline-closure, the FIRST story in Epic 21 — multi-story epic, 6 stories) ... PE.02–PE.04 sizing decisions hard-depend on this story's output ... AP18-C2 atomic patch held."* Story file likely exists at `implementation-artifacts/21-1-k6-baseline-closure.md` (created 2026-05-04 in same commit as the epic-21 transition). Status `ready-for-dev`.

**Risk:** dispatching `bmad-dev-story` for 21-1 without first running pre-Epic-21 IR-v6 would be the **fourth consecutive epic** to bypass the pre-epic IR gate (CR-10). Per epic-20 retro elevations (CR-11), Story 21-1 also has a hard precondition: an actual k6 run + commit must land before the story can complete its baseline-closure premise. **The single most important gate v21 names: dispatch IR-v6 NOW, before 21-1 dev.**

---

## Operative routing — REPLACES IR-v5 §Recommended Next Steps

IR-v5's eight-step list is now substantially obsolete (2/8 discharged, 4/8 not discharged but with materially different urgency and scope post-E18/19/20-close, 1/8 partially discharged, 1/8 operationally moot). v21 names the new operative dispatch order:

### Immediate (next 24–48 hours)

1. **🛑 CRITICAL — Dispatch `bmad-check-implementation-readiness` scoped to Epic 21 (IR-v6).** Validates: (a) E21 epic file is internally consistent with PRD §NFR reliability targets and `architecture.md` ADR-010; (b) the 6-story breakdown (21-1..21-6) covers all reliability dimensions without overlap with E12 analytics observability; (c) Story 21-1 (k6-baseline-closure) is genuinely a follow-on of an existing harness rather than a fresh build; (d) PostgreSQL HA (21-2) is feasible against current single-postgres infra (production topology decision required); (e) Redis HA (21-3) does not regress AP15-04 Redis-stream consumer patterns or AP19-CSM-3 alert.created publish path. **Effort: ~30–60 min.** Gates Story 21-1 dev dispatch.

2. **🛑 CRITICAL — Dispatch `inj-01-dependabot-configuration` (Epic 13 carry-forward, 17 epic boundaries late).** Per epic-20 retro Critical-prep-before-E21 list. Single file (`.github/dependabot.yml`); <30 min. After landing, retroactively scan all dep additions from E18 + E19 + E20 stories and create follow-up advisories (likely `dw-04` or per-story amendments) for any findings.

3. **🛑 HIGH — Run k6 baseline + commit results.** Per epic-20 retro Critical-prep list. This is the precondition for Story 21-1's baseline-closure premise. Without committed k6 numbers in-repo, 21-1 cannot complete its own AC ("close the k6 baseline carry-forward"). Coordination question for operator: is this dispatched as part of Story 21-1 itself, or as a precondition step before 21-1 dev?

### After IR-v6 lands

4. **`[VS] Validate Story` for Story 21-1.** Non-negotiable per Operator workflow guidance.
5. **`bmad-dev-story` for Story 21-1.** Two-gate enforcement (AP17-C1 mitigation sustained — 7-story streak target).
6. **`bmad-code-review` for Story 21-1.** Pass-2 Approve required for `done` transition.

### After Story 21-1 closes Approved

7. **Sequential or parallel dispatch of Stories 21-2 (PostgreSQL HA), 21-3 (Redis HA), 21-4 (PodDisruptionBudgets).** Per epic-20 retro pickup rationale (line 50): "Story 21-2 (PostgreSQL HA) and 21-3 (Redis HA) require infrastructure-topology decisions ... that should be informed by 21-1 baseline numbers." Sequencing: 21-2 + 21-3 + 21-4 can run in parallel after 21-1 closes.
8. **Story 21-5 (SLO dashboards + error-budget alerting) after 21-1 + 21-2 + 21-3 + 21-4.** Hard dependency on baseline numbers (21-1) and HA topology (21-2/3) and budget enforcement (21-4).
9. **Story 21-6 (on-call rotation + runbooks) after 21-5.** Runbooks reference SLO dashboards.

### Carry-forward debt (non-blocking on E21 but accumulating)

10. **Retroactive `bmad-code-review` re-passes for 17-0 / 17-1 / 17-2 / 17-3 / 18-0 (CR-9).** Original IR-v4 / IR-v5 recommendation — five stories `done` without explicit Approve verdict. Audit-trail liability before any future GA-gate or external compliance audit. Sequencing: chronological (17-0 first). Effort: 30–45 min × 5.
11. **CR-1 + CR-2 + CR-4 batched hygiene pass.** ~30–60 min operator-level decision. **Should land before Epic 22 kickoff** to prevent further epic-state-of-truth drift. Recommended NOT to block Epic 21 on this.
12. **AP18-C1 18-3-trust-center-hardening injection.** Open backlog injection candidate per epic-18 retro. Resolves WeasyPrint D6/D7 deviations from Story 18-1. **Operationally non-urgent** — E19 / E20 / E21 do not surface trust-center pages. Operator-discretion injection.
13. **Status-header anti-pattern (AP18-C2 → AP19-C1 → AP20-C1; 18 consecutive epics).** Codified but not yet structurally remediated (project-context rule + dev-story-template update path is the established mitigation; epic-20 retro line 337 names "dev-story template updated with 5 new anti-patterns" as Critical-prep-before-E21).

### Process-layer recommendations (orchestrator dispatcher; v21 scope-out)

14. **Orchestrator dispatcher 24h-suppression-window for autonomous-drift-detection trigger** when a `sprint-change-proposal-*` OR `implementation-readiness-report-*` has been generated within the last 24 hours. **21st-fire pattern with verbatim brief is the operative meta-finding.** Same recommendation as v18 §Approval meta-recommendation (b), v19 restatement, v20 §The 20th-fire meta-pattern. Out-of-scope for `bmad-correct-course`; operator routing-layer responsibility.

---

## What v21 is *not*

- v21 is **not** an artefact-edit proposal. Zero PRD / architecture / epics / ux-spec / story-file edits.
- v21 is **not** a sprint-status edit proposal. Per project memory rule: *"sprint-status.yaml is orchestrator-managed — surgical edits only."* No surgical edit is warranted (v21 records no story state change).
- v21 is **not** a generation of IR-v6. `bmad-correct-course` is not the IR-generation skill. v21 routes the operator to `bmad-check-implementation-readiness` as the next-skill gate.
- v21 is **not** a re-press of v20 / IR-v5's open recommendations. The 4/8 not-discharged items are reformulated above with materially different urgency / scope post-E18/19/20-close.
- v21 is **not** a supersession of v20 or IR-v5. Both remain on the audit trail as the operative reference for the historical state at their respective fire times. v21 is the operative reference for state from 2026-05-04 ~22:00 forward, until IR-v6 supersedes it.

---

## The 21st-fire meta-pattern

The 21st-fire continues the established orchestrator dispatcher pattern:

- **v19 finding:** dispatcher does not consult prior `sprint-change-proposal-*.md` files.
- **v20 generalisation:** dispatcher does not consult `implementation-readiness-report-*.md` files either.
- **v21 confirmation:** dispatcher pattern persists across 19+ hours and three full epic boundaries (E18 close, E19 open+close, E20 open+close, E21 entry). The brief is identical and the trigger is autonomous.

**No new action at the `bmad-correct-course` layer is recommended.** The mitigation remains the same as v18 §Approval meta-recommendation (b) and v19/v20 restatements: orchestrator routing-layer 24h-suppression-window when a sprint-change-proposal OR implementation-readiness-report exists in the last 24h. v21 defers this to operator routing-layer responsibility unchanged from prior versions.

---

## Approval

**Scope classification: MINOR.** Zero artefact edits. Zero status transitions. Two new audit-trail entries: (a) "v21 fired 2026-05-04 ~22:00 post-IR-v5; routes to IR-v6 as next-skill gate; records state advance through 3 epic closures + E21 entry"; (b) "CR-10 + CR-11 added (pre-epic IR gate violated for 3 consecutive epics; AP20-C2 k6 absence is HARD BLOCK on Story 21-1)".

**Auto-approved per autonomous BMAD autopilot mode** (no operator prompts; no menus; no input expected).

**Handoff:** v21's single highest-priority operative dispatch — `bmad-check-implementation-readiness` scoped to Epic 21 — is the next-skill target. Secondary parallel dispatch: `inj-01-dependabot-configuration` (Epic 13 carry-forward, 17 epic boundaries late, on epic-20 retro Critical-prep-before-E21 list).

---

## Workflow execution log entry

```yaml
date: 2026-05-04
workflow: bmad-correct-course
mode: BMAD autopilot (no operator prompts)
trigger_brief: "Drift detected. Review requirements." (verbatim, 21st consecutive fire)
input_artifacts:
  - planning-artifacts/sprint-change-proposal-2026-05-04-v2.md (v20, 02:55 — post-IR-v4 no-op)
  - planning-artifacts/implementation-readiness-report-2026-05-04-v2.md (IR-v5, 03:16 — reaffirms IR-v4)
  - implementation-artifacts/sprint-status.yaml (last touched 2026-05-04 22:03; latest entries: epic-18/19/20 done; epic-21 in-progress; story 21-1 ready-for-dev; story 19-1 review)
  - planning-artifacts/project-context.md
  - implementation-artifacts/epic-18-retro-2026-05-04.md
  - implementation-artifacts/epic-19-retro-2026-05-04.md
  - implementation-artifacts/epic-20-retro-2026-05-04.md
output_artifacts:
  - planning-artifacts/sprint-change-proposal-2026-05-04-v3.md (this file)
edits_to_existing_artifacts: 0
status_transitions: 0
operative_routing: "Dispatch bmad-check-implementation-readiness scoped to Epic 21 (IR-v6) BEFORE Story 21-1 enters bmad-dev-story. Parallel: inj-01 Dependabot dispatch."
net_new_findings:
  - CR-10 (CRITICAL): pre-epic IR gate violated for 3 consecutive epics (E19, E20, E21-entry)
  - CR-11 (CRITICAL): AP20-C2 k6 absence is HARD BLOCK on Story 21-1 baseline-closure
  - 21st-fire dispatcher pattern persists across 3 epic closures and E21 entry (~19h since IR-v5)
scope_classification: Minor
approval: auto-approved (BMAD autopilot)
handoff: bmad-check-implementation-readiness (next-skill); inj-01-dependabot-configuration (parallel)
```

---

*Generated by `bmad-correct-course` skill in BMAD autopilot mode (no operator prompts) on 2026-05-04. Unlike v20 (which was a deliberate near-no-op post-IR-v4), v21 records substantial state advance (3 epic closures + E21 entry in the v20→v21 window) and re-routes the operative dispatch chain to a fresh `bmad-check-implementation-readiness` (IR-v6) scoped to Epic 21. The single highest-priority operative item in the project is now IR-v6 dispatch (replacing IR-v5 §Recommended Next Steps #1, which was discharged via 18-1 Pass-2 Approve).*
