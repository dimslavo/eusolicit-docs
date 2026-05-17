---
Status: advisory (PM-authored dispatch sequencing for orchestrator)
Epic: E22 + E23 launch-closeout cluster
Author: 📋 John (PM)
Created: 2026-05-15 (via IR remediation — Issues #3 + #4)
Source: `implementation-readiness-report-2026-05-15.md`; on-disk audit of `sprint-status.yaml` + each story file
---

# PM Audit Memo — E22 / E23 Dispatch Sequencing

## Purpose

The IR report flagged E22 (6 stories at `review`, 0 `done`) and E23 (4 of 5 stories with gaps) as 🔴 launch-blocking. This memo gives the orchestrator a sequenced dispatch plan, story-by-story, with the verified on-disk state of each. No code changes — pure planning.

## E22 — On-Prem Launch

### Current state (verified 2026-05-15 from sprint-status + story files)

| Story | Status | Code-complete? | Gate to `done` |
|---|---|---|---|
| onprem-01-postgres-backup-and-recovery | `review` | ✅ Yes | `bmad-code-review` Approve + operator weekly-restore-test drill |
| onprem-02-redis-persistence-and-recovery | `review` | ✅ Yes | `bmad-code-review` Approve + operator controlled-restart drill |
| onprem-03-monitoring-on-www1 | `review` | ✅ Yes | `bmad-code-review` Approve + operator decision (reuse vs standalone observability stack) |
| onprem-04-disk-and-resource-monitoring | `review` | ✅ Yes | `bmad-code-review` Approve + first synthetic alert verified |
| onprem-05-deploy-guardrails | `review` | ✅ Yes | `bmad-code-review` Approve + branch-protection visually confirmed in GitHub UI |
| onprem-06-www1-itself-as-code | `review` | ✅ Yes | `bmad-code-review` Approve + practice rebuild on sacrificial VM (AP22-D1 discipline) |
| onprem-07-n8n-instance-ops | `backlog` | ❌ NEW 2026-05-15 | story dispatch → code → review → Approve + sacrificial-VM rebuild drill |

### Recommended dispatch sequence

**Phase 1 — bmad-code-review cascade (5 review passes, parallelizable):**

The 6 `review` stories can run reviews in parallel because they don't share code-review surface. Recommend invoking `bmad-code-review` on each in this order based on launch-risk:

1. **`onprem-01-postgres-backup-and-recovery`** — sequence-first. RPO foundation; everything downstream depends on Postgres backup being trustworthy. Don't ship anything else until this is Approve.
2. **`onprem-02-redis-persistence-and-recovery`** — RTO for ephemeral state (Celery + Redis Streams). Run in parallel with onprem-01 review.
3. **`onprem-05-deploy-guardrails`** — closes the auto-sync outage class (`project_auto_sync_quality.md` memory). Higher-priority for launch posture than monitoring stories because it prevents the failure class that triggered the on-prem pivot.
4. **`onprem-03-monitoring-on-www1`** — needed before pe-04 chaos drill (E23) and onprem-04 alert verification.
5. **`onprem-04-disk-and-resource-monitoring`** — depends on onprem-03 Alertmanager being live to actually deliver alerts. Run review after onprem-03 Approve.
6. **`onprem-06-www1-itself-as-code`** — RTO defence story; needed before any production rebuild is honest. Can run review in parallel with 03/04.

**Phase 2 — operator drills (in story file §Drill Results):**

After each story reaches Approve, the operator-side drill must execute and populate the story file's §Drill Results section. These are wall-clock activities, not orchestrator dispatchable:

| Story | Drill | Estimated wall-clock |
|---|---|---|
| onprem-01 | Full restore-to-fresh-volume from Hetzner Storage Box | 2-4h (one drill window) |
| onprem-02 | Controlled-restart verifying all 6 services reconnect + Streams offsets survive | 30 min |
| onprem-03 | Reuse-vs-standalone decision documented; 7 Grafana dashboards loaded; 4 rule files active | 1h verification |
| onprem-04 | First synthetic page on each host alert; runbook_url annotations validated | 1h |
| onprem-05 | Branch-protection visible in GitHub UI; `chore: auto-sync` PR policy tested | 30 min |
| onprem-06 | Sacrificial Debian 13 VM rebuild (Hetzner Cloud trial or local KVM); §First Rebuild Drill Results populated | 4-6h (one drill window) |
| onprem-07 | Sacrificial VM N8N rebuild + restore + upgrade + rollback (when dispatched) | 4h |

**Phase 3 — onprem-07 dispatch:**

New story (added 2026-05-15). Stand-alone — no upstream blocker; can be dispatched immediately or after Phase 1+2 completes. Recommendation: defer to post-launch sprint to keep current launch cadence focused.

**Phase 4 — E22 epic transition `in-progress → done`:**

When 6/6 (onprem-01..06) reach `done` AND the E22 epic AC for N8N is either met (onprem-07 done) OR explicitly deferred via a Known Deviation entry in the epic file. Per BMAD rule: epic-state flip is manual once all stories reach done.

---

## E23 — Operational Close-Out

### Current state (verified 2026-05-15)

| Story | Status | Code-complete? | Gate to `done` |
|---|---|---|---|
| drift-recovery-story | `done` (2026-05-13) | ✅ | — closed |
| inj-03-tea-review-backlog-epic8-epic9 | `done` (2026-05-13) | ✅ | — closed |
| pe-06-pagerduty-rotation-provisioning | `done` (per sprint-status value, but comment says "review pending bmad-code-review Approve + operator alert-delivery test") | ⚠️ status/comment mismatch | reconcile: re-check actual delivery test result; either value→`review` or update comment to remove the pending-language |
| pe-04-chaos-drill-execution | `ready-for-dev` | ⚠️ PARTIAL — runbook missing redis AOF drill (3/4 drills specified) | Author redis AOF drill in runbook → dispatch bmad-dev-story → bmad-code-review → operator-paced chaos drills (4 drill windows) |
| public-sla-announcement-soak-gate | `ready-for-dev` | ⚠️ PARTIAL — MDX cards landed, but i18n keys missing from both en.json AND bg.json | Author i18n keys → `pnpm check:i18n` green → bmad-dev-story → bmad-code-review → product+legal sign-off → 2-week soak (calendar gate) |
| sirmaai-eu-residency-due-diligence | `backlog` | ❌ NEW 2026-05-15 | PM + legal work; story dispatch only after SirmaAI DPA addendum signed |

### Recommended dispatch sequence

**Step 1 — reconcile pe-06 status/comment mismatch (PM hygiene, 5 min):**

sprint-status.yaml line 406 has `pe-06-pagerduty-rotation-provisioning: done` but the trailing comment says *"Flipped ready-for-dev → review pending bmad-code-review Approve + operator alert-delivery test"*. Either:
- (a) Value is correct (done) — update the comment to remove the "pending" language.
- (b) Comment is correct (still review) — flip the value back from done to review.

PM recommendation: verify on www1 whether the test synthetic page has actually reached email + Telegram inboxes. If yes → keep value=done, update comment. If no → flip value to review, run the test, then flip to done after Approve.

**Step 2 — close pe-04 (chaos drill execution):**

Sub-sequence:
1. Author the missing redis AOF drill section in `eusolicit-docs/runbooks/chaos-drill-single-host.md`. Mirror the structure of the 3 existing drills (container kill, disk fill, postgres crash). ~30 min PM work, or could be a small dev-story dispatch.
2. Re-verify story file ACs cover all 4 drills now.
3. Dispatch `bmad-code-review` on the runbook PR for AC1-3 closure.
4. Operator executes all 4 drills sequentially (recommend one per week to spread cognitive load; or single 1-day drill window if launch pressure demands).
5. Post-mortem authored using `eusolicit-docs/incident-management/post-mortem-template.md` per AC.
6. Story flips ready-for-dev → review → done with two-gate close.

**Step 3 — close public-sla-announcement-soak-gate:**

Sub-sequence:
1. Add the two missing i18n keys to both `messages/en.json` AND `messages/bg.json`:
   - `trust.posture.availability.description`
   - `trust.posture.rtoRpo.description`
   Maintain 1799-key parity. PM or quick dev pass (15 min); easier to bundle into `bmad-dev-story` pass.
2. Verify `pnpm check:i18n` green.
3. Dispatch `bmad-code-review` for AC1-4 + AC6-7 closure.
4. Product + legal sign-off on the disclosure copy (AC7) — this is content review, not code review.
5. Flip ready-for-dev → review → done.
6. The 2-week soak gate is wall-clock dependent — AC5 was explicitly split to a follow-up story (`public-sla-7-day-soak-observation`) per the 2026-05-13 sprint change proposal. Don't block this story's close on the soak observation.

**Step 4 — initiate sirmaai-eu-residency-due-diligence (NEW):**

PM-led, not code work:
1. Deb (PM) opens DPA conversation with SirmaAI commercial / legal contact.
2. Legal counsel reviews resulting addendum against AC1 (a)/(b)/(c).
3. PM lands AC3-6 as one PR (legal-docs archive + Trust Center disclosure + PRD reconciliation + project-context update).
4. `bmad-code-review` Approve on the doc/MDX PR; operator flips to done after the legal addendum is signed.
5. **Risk if blocked:** if SirmaAI cannot honour residency clauses within commercial timeframe, escalate to sprint-change-proposal — either renegotiate launch date or architecture amendment for self-hosted SirmaAI fleet (out of scope for current launch posture).

**Step 5 — E23 epic transition `in-progress → done`:**

When all 6 stories reach done. The 2-week soak gate (separate `public-sla-7-day-soak-observation` story) is post-launch and doesn't block E23 epic closure if it's been split.

---

## Cross-epic launch gate

E22 epic transition done + E23 epic transition done + 2-week post-launch soak (`public-sla-7-day-soak-observation`) elapsed are the three gates to flip the platform's public posture from "Beta — best effort" to whatever commercial commitment comes next. None of those steps requires further planning artefacts beyond what's described above and in the IR report.

## Estimated wall-clock to all `done`

Assuming parallel operator availability + orchestrator dispatch capacity:

| Phase | Estimate |
|---|---|
| E22 bmad-code-review cascade (6 reviews in parallel) | 1-2 days |
| E22 operator drills (onprem-01..06) | 2-3 days total drill time across 1-2 weeks |
| E23 pe-04 + public-sla close | 2-3 days |
| E23 pe-06 reconciliation | 1 hour |
| E23 sirmaai-eu-residency (legal cycle) | 1-2 weeks wall-clock |
| Cross-epic gate (E22+E23 done) | ~2 weeks total |
| Post-launch 2-week soak | calendar — separate from launch readiness |

**Earliest credible launch readiness gate clearance:** 2026-05-29 (~2 weeks from today) — **but only if** sirmaai-eu-residency-due-diligence is responded to quickly by SirmaAI. If SirmaAI's legal cycle is long, launch slips.

## Open coordination items

- **Operator allocation for drills.** onprem-01 (full restore) + onprem-06 (sacrificial VM rebuild) + pe-04 (4 chaos drills) collectively need ~3 dedicated drill days from platform-engineering. If allocation is the bottleneck, sequence them across a 2-week window.
- **Status-page domain.** UX amendment §8 open question 3 mentions `https://status.eusolicit.com/sirmaai` as a candidate link for the degraded-mode banner — confirm whether this domain exists / will be provisioned.
- **SirmaAI commercial contact.** Who at SirmaAI does PM (Deb) reach for the DPA addendum? Needs to be identified before sirmaai-eu-residency-due-diligence can dispatch effectively.

---

**Cross-references:**
- IR report: `implementation-readiness-report-2026-05-15.md`
- E22 epic: `epics/E22-onprem-launch.md` (AC list bumped 2026-05-15 with N8N invariant)
- E23 epic: `epics/E23-operational-close-out.md` (story count bumped 5→6; closure signal added for residency)
- New story: `sirmaai-eu-residency-due-diligence.md`
- New story: `onprem-07-n8n-instance-ops.md`
- UX amendment: `ux-spec-amendment-2026-05-15-sirmaai.md`
- E26 baseline spec: `test-artifacts/e26-baseline-datasets-spec.md`
