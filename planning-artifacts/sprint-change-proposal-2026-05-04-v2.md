---
date: 2026-05-04
generator: bmad-correct-course (BMAD autopilot, no operator prompts)
trigger_brief: "Drift detected. Review requirements." (verbatim, identical to v2–v19; **20th consecutive fire**)
project: EU Solicit
scope_classification: "Process / sequencing — Minor (defer to IR-v4)"
supersedes: ""  # does NOT supersede v19; v19 + IR-v4 remain operative
defers_to: "implementation-readiness-report-2026-05-04.md (IR-v4, 2026-05-04 02:17)"
relationship_to_prior:
  - "v19 (2026-05-04 01:56) — operative routing for forward dispatch (18-1 code-review pass-1)"
  - "IR-v4 (2026-05-04 02:17) — operative routing for everything else; supersedes v19 for retroactive AP17-C1 closure ordering, CR-6 elevation, and CR-7 sequencing"
  - "v20 fires <45 minutes after IR-v4; <3 hours after v19. v18 §Approval meta-recommendation (b) suppression-window remains unenforced at the orchestrator dispatcher."
no_artifact_edits: true   # no PRD / architecture / epics / ux-spec edits proposed
no_sprint_status_edits: true  # sprint-status.yaml is orchestrator-managed (project-context rule)
---

# Sprint Change Proposal — 2026-05-04 (v20, post-IR-v4 / 20th-fire / no-new-action)

## TL;DR

**No new substantive findings since v19 (2026-05-04 01:56) or IR-v4 (2026-05-04 02:17).** Sprint-status was last touched 2026-05-04 02:41 with the v19-emission entry only — no story-state transitions, no code-review verdicts, no Sally dispatch, no `inj-01` dispatch in the ~45-minute window before v20 fired. **All operative routing recommendations remain those enumerated in IR-v4 §Recommended Next Steps (1–8) and §CR Summary Table.** v20 records the 20th-fire pattern as the only net-new signal and defers to IR-v4.

**This proposal proposes zero artefact edits.** No PRD, architecture, epics, ux-spec, or story-file changes. No sprint-status transitions (orchestrator-managed; surgical-edits-only per project memory rule). v20 is a routing **acknowledgement**, not a routing **redirect**.

---

## Why this proposal exists

The `bmad-correct-course` workflow was dispatched for the 20th consecutive time with the verbatim brief *"Drift detected. Review requirements."* The orchestrator dispatcher does not consult prior `sprint-change-proposal-*.md` files or the most recent `implementation-readiness-report-*.md` as state before firing. v19 named this dispatcher-bias explicitly (v19 §Operative observation). IR-v4 named it again (IR-v4 §Executive Verdict ¶3 — *"v19 also names the 19th-fire-in-<24h pattern itself as the operative meta-finding (orchestrator dispatcher consuming the verbatim drift brief without consulting prior proposals as state)."*). v20 fires <45 minutes after IR-v4 closed and confirms the dispatcher remains unaware of IR-v4.

**v20's job is not to re-litigate.** Re-pressing recommendations the dispatcher has empirically rejected 19 times is itself an established anti-pattern (v17/v18/v19 all noted this rationale verbatim). v20's job is to:

1. Acknowledge IR-v4 as the now-canonical source for dispatch routing (supersedes v19 where they differ).
2. Record the 20th-fire pattern as a single line in the audit trail.
3. Produce zero new edit proposals, zero status transitions, zero file rewrites.

---

## Substantive delta vs v19 + IR-v4

| Dimension | State at v19 close (01:56) | State at IR-v4 close (02:17) | State at v20 fire (~03:00–04:00) | Delta |
|---|---|---|---|---|
| Story 18-1 | `review`, awaiting `bmad-code-review` pass-1 | unchanged | unchanged | **No code-review dispatch in ~45-min window after IR-v4.** |
| Story 18-0 | `done` w/o Approve (AP17-C1 #5) | unchanged | unchanged | — |
| Stories 17-0/17-1/17-2/17-3 | `done` w/o Approve (AP17-C1 #1–#4) | unchanged | unchanged | — |
| Sally pre-pass for E18 (CR-7) | not dispatched (6 working days aged) | unchanged | unchanged | **No Sally dispatch in ~45-min window.** |
| `inj-01-dependabot-configuration` (CR-6 / R-018-4) | `ready-for-dev`; first-concrete-harm marker | unchanged | unchanged | **Mandatory pre-18-2 per IR-v4; not dispatched.** |
| CR-1 / CR-2 / CR-4 hygiene | unresolved (9 working days aged) | unchanged | unchanged | — |
| sprint-status.yaml | last_updated 2026-05-04 (v19 emission entry, line 76) | unchanged | unchanged (last touch 02:41 = v19 entry) | **No new story state transitions.** |
| project-context.md | last touched 2026-05-03 (Epic 17 retro entries) | unchanged | unchanged | — |
| Latest IR | IR-v3 (2026-05-03) | **IR-v4 (2026-05-04 02:17)** — supersedes IR-v3 for E18 mid-epic readiness | unchanged | **IR-v4 is now the operative readiness reference.** |

**Net delta in the v19 → v20 window: zero state transitions.** The only file change is sprint-status.yaml line 76 (the v19-emission self-record) at 02:41, which is an artefact of the v19 invocation itself — not a state-of-the-world change.

---

## Operative routing — defer to IR-v4

v20 explicitly defers to **IR-v4 §Recommended Next Steps (1–8)** and **IR-v4 §CR Summary Table**. The dispatch order, recapped here for traceability only:

1. **Forward `bmad-code-review` for Story 18-1 (pass-1).** Single in-flight item with clean dispatch path. Verdict MUST be explicit Approve/Changes-Requested/Reject and recorded in story file §7. (IR-v4 §Critical Issues #1; v19 §Path forward Action 1.)
2. **Parallel: `bmad-agent-ux-designer` (Sally) for E18 6 surfaces (CR-7).** ~half-day; gates `bmad-create-story` for 18-2; resolves R-018-2/3/5 inline. (IR-v4 §Critical Issues #2; v19 §Path forward Action 3.)
3. **Parallel: `inj-01-dependabot-configuration` (CR-6 / R-018-4).** <30 min; mandatory pre-18-2 per IR-v4 elevation (was "recommended" in v19 / IR-v3). (IR-v4 §Critical Issues #4.)
4. **After 18-1 closes Approved: sequential retroactive `bmad-code-review` re-passes for 17-0 → 17-1 → 17-2 → 17-3 → 18-0 (CR-9 / AP17-C1 quintuple-fire closure).** ~150–225 min total. (IR-v4 §Critical Issues #3; v19 §Path forward Action 2.)
5. **Operator-level batched hygiene pass (CR-1 + CR-2 + CR-4) before Epic 19 kickoff.** ~30–60 min. (IR-v4 §Critical Issues #5–#7.)
6. **Epic 18 close-out: `[ER] Epic Review` then `bmad-retrospective` for Epic 18.** (IR-v4 §Recommended Next Steps #7.)
7. **Pre-Epic-19: `[IR] Implementation Readiness` v5 (non-negotiable per Operator BMAD-stream).** (IR-v4 §Recommended Next Steps #8.)

**v20 adds no items to this list and reorders nothing.** IR-v4's ordering is sound and 45 minutes old.

---

## What v20 is *not*

- v20 is **not** a re-press of v19's three findings. v19 stands.
- v20 is **not** a supersession of v19. v19 still routes the forward 18-1 dispatch with its enumerated reviewer brief (v19 §Path forward Action 1 line-by-line per-AC enumeration is operative for the eventual `bmad-code-review` reviewer).
- v20 is **not** a parallel/alternative path. IR-v4 already chose Path-A and Option-A is established.
- v20 does **not** edit sprint-status.yaml. Per project memory: *"sprint-status.yaml is orchestrator-managed — don't let BMAD sprint-planning regenerate it; surgical edits only."* No surgical edit is warranted here (no story state changed).
- v20 does **not** edit PRD / architecture / epics / ux-spec. None of the operative blockers are spec-incoherence; they are dispatch and process-integrity.
- v20 does **not** generate IR-v5. `bmad-correct-course` is not the IR-generation skill; IR-v5 is owed pre-Epic-19 per `bmad-check-implementation-readiness`.

---

## The 20th-fire meta-pattern (single net-new audit-trail entry)

The only signal v20 contributes that v19 / IR-v4 did not already record is the **continuation of the dispatcher pattern past IR-v4**. Specifically:

- IR-v4 closed 2026-05-04 02:17 with explicit dispatch ordering and a §HALT determination of "No HALT."
- The orchestrator dispatcher fired `bmad-correct-course` again ≤45 minutes later with the same verbatim brief.
- This means **the dispatcher does not consult `implementation-readiness-report-*.md` either** — not just prior `sprint-change-proposal-*.md` files (v19 finding) but also IR documents.

This generalises the v19 §Operative observation. The mitigation is unchanged from v18 §Approval meta-recommendation (b) and v19's restatement: **suppress autonomous-drift-detection trigger when a sprint-change-proposal OR implementation-readiness-report has been generated within the last 24 hours.** This is an orchestrator routing-layer change, not a `bmad-correct-course` deliverable.

**v20 records this generalisation. No further action is taken at this layer.**

---

## Approval

**Scope classification: MINOR.** Zero artefact edits. Zero status transitions. Single audit-trail line: "v20 fired 2026-05-04 post-IR-v4; defers to IR-v4 routing; records 20th-fire-also-post-IR pattern."

**Auto-approved per autonomous BMAD autopilot mode** (no operator prompts; no menus; no input expected).

**Handoff:** None. The single highest-priority operative dispatch in the project — `bmad-code-review` for Story 18-1 (pass-1) — is unchanged from v19 and is the operator/orchestrator's existing handoff target. v20 takes no new handoff action.

**Out-of-scope items (all carried forward verbatim from v19 / IR-v4 §Recommended Next Steps; v20 takes no position):**

1. CR-7 Sally dispatch (IR-v4 critical #2).
2. CR-9 retroactive AP17-C1 5-pass closure (IR-v4 critical #3).
3. CR-6 / `inj-01` dispatch (IR-v4 critical #4).
4. CR-1 + CR-2 + CR-4 batched hygiene pass (IR-v4 high #5–#7).
5. CR-8 design-question resolution (IR-v4 medium #8).
6. CR-5 E17 backlog tech debt (IR-v4 low #9).
7. Epic 17 retrospective action-item dispatch (13 items per `epic-17-retro-2026-05-03.md`).
8. Orchestrator dispatcher 24h-suppression-window for autonomous-drift-detection trigger (the 20th-fire meta-finding).

---

## Workflow execution log entry

```yaml
date: 2026-05-04
workflow: bmad-correct-course
mode: BMAD autopilot (no operator prompts)
trigger_brief: "Drift detected. Review requirements." (verbatim, 20th consecutive fire)
input_artifacts:
  - planning-artifacts/sprint-change-proposal-2026-05-04.md (v19, 01:56)
  - planning-artifacts/implementation-readiness-report-2026-05-04.md (IR-v4, 02:17)
  - implementation-artifacts/sprint-status.yaml (last touched 02:41, v19-emission entry only)
  - planning-artifacts/project-context.md (unchanged since 2026-05-03)
output_artifacts:
  - planning-artifacts/sprint-change-proposal-2026-05-04-v2.md (this file)
edits_to_existing_artifacts: 0
status_transitions: 0
operative_routing: defer to IR-v4 §Recommended Next Steps (1–8)
net_new_finding: 20th-fire-also-post-IR pattern (orchestrator dispatcher does not consult IR documents either)
scope_classification: Minor
approval: auto-approved (BMAD autopilot)
handoff: none (existing IR-v4 dispatch chain is unchanged)
```

---

*Generated by `bmad-correct-course` skill in BMAD autopilot mode (no operator prompts) on 2026-05-04. v20 is a deliberate near-no-op; it acknowledges the 20th-fire pattern, names IR-v4 as the canonical dispatch reference, and proposes zero substantive edits. The single highest-priority operative item in the project remains `bmad-code-review` for Story 18-1 (pass-1), unchanged from v19 §Path forward Action 1 and IR-v4 §Recommended Next Steps #1.*
