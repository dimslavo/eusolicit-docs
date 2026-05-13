# Story: pe-04-chaos-drill-execution

**Epic:** E23 — Operational Close-Out
**Status:** ready-for-dev
**Story key:** `pe-04-chaos-drill-execution`
**Type:** ops / operator-paced execution
**Sprint:** E23 close-out
**Created:** 2026-05-13 by `bmad-create-story` (📋 John PM dispatch under `proceed with all` autopilot) per sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md §3.1.2 + §4.2 (approved by Deb 2026-05-13)

> **Provenance** — this is a re-create of the pe-04 story spec after the 2026-05-13 audit surfaced that the sprint-status row had been injected at `ready-for-dev` without a formal story file at `story_location` (E22-AP01 / E23-AP02 anti-pattern recurrence flagged in the 2026-05-12 retros). Pull-back to `backlog` landed via Edit 1 of the proposal; this `bmad-create-story` dispatch lands the spec and flips to `ready-for-dev`.

## Goal

Execute the four post-pivot chaos drills against the single-host Docker deployment on `www1` and produce verified evidence (recovery times, post-mortems, alert-routing confirmation) that the platform's compensating-control posture under ADR-010 (no horizontal HA, rehearsed manual recovery) actually works as designed. Story is **operator-paced** — the dev agent's job is to assist with runbook completion, post-mortem authoring, and AC verification; the operator runs the actual drills at a low-traffic window.

**Scope (4 drills per the approved proposal):**

1. **Postgres failover** — verify pg WAL replay + RPO ≤ 15min per onprem-01 backup posture
2. **Redis controlled-restart** — verify AOF replay + Streams consumer-group offset preservation; reconnect window ≤ 30s
3. **Container restart-loop** — verify `ContainerRestartLoop` Prometheus rule (onprem-04 host-alerts.yaml) fires when prometheus + cadvisor + grafana are killed
4. **Network partition** — verify circuit-breaker open + degraded-mode banner (E28 S28.07) when egress to SirmaAI (`agenticsai.endigitalx.com`) is blocked

**Out of scope:**
- **Decomposition into 4 sibling stories** — operator can split later if drills run across multiple sprint windows. PM recommendation: keep monolithic unless the first run reveals dispatch-window pressure (see §Risk).
- The runbook's existing "disk fill" drill (Drill 2 in `runbooks/chaos-drill-single-host.md`) — useful supplementary drill but NOT one of the 4 approved ACs (disk-fill alerting is already verified by `HostRootDiskWarning` + `HostRootDiskCritical` in host-alerts.yaml per sibling story onprem-04). May be kept in the runbook as a bonus drill or archived during AC6 runbook completion.
- Kubernetes-flavoured chaos primitives from the archived pre-pivot runbook (`implementation-artifacts/.archive/pe-04-chaos-drill-runbook.md`) — SUPERSEDED by ADR-010 single-host posture.

---

## Acceptance Criteria

### AC1 — Postgres failover drill executed; RPO measured

- [ ] Drill 3 (postgres crash recovery) per `runbooks/chaos-drill-single-host.md` executed against staging or sacrificial window on www1
- [ ] Pre-drill sentinel row written to `shared.audit_log` and verified to survive WAL replay post-restart
- [ ] Recovery time measured: postgres `healthy` status restored within ≤ 60s of kill
- [ ] All 6 application services (`client-api`, `admin-api`, `data-pipeline`, `ai-gateway`, `notification`, `integrations-api`) reconnect via `pool_pre_ping` without manual intervention
- [ ] RPO calculation documented: time between last successful WAL segment archive (per onprem-01 archive_timeout=60 + 15-min off-site push) and the drill kill moment — target ≤ 15min off-site, near-zero local
- [ ] Drill outcome row appended to §Drill Results table in `runbooks/chaos-drill-single-host.md`
- [ ] Post-mortem committed at `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-postgres.md` (date = drill date; template = existing `post-mortems/2026-MM-DD-pe-04-chaos-drill.md` placeholder)

### AC2 — Redis controlled-restart drill executed; reconnect window measured

- [ ] Drill 4 (redis AOF replay) per `runbooks/chaos-drill-single-host.md` executed
- [ ] Pre-drill sentinel key (`chaos-sentinel`) written and verified to survive AOF replay post-restart
- [ ] Streams consumer-group offsets preserved (verified via `XINFO STREAMS eusolicit.events`)
- [ ] Reconnect window measured: redis-unavailable duration ≤ 30s (Celery + redis-py keepalive reconnect)
- [ ] All 6 services healthy on `/healthz` post-recovery
- [ ] Drill outcome row appended to §Drill Results table in `runbooks/chaos-drill-single-host.md`
- [ ] Post-mortem committed at `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-redis.md`

### AC3 — Container restart-loop drill (prometheus + cadvisor + grafana)

- [ ] All three observability containers killed in quick succession via `docker kill` (≥3 restarts within 5min per `ContainerRestartLoop` PromQL rule at `host-alerts.yaml:81-91`)
- [ ] `ContainerRestartLoop` alert fires; severity=page; routed via pe-06 `page-email-and-telegram` receiver (sanity-check against pe-06 routing closed 2026-05-13 — see AC7)
- [ ] Auto-restart succeeds for all three (`restart: unless-stopped` policy verified)
- [ ] Operator confirms email + Telegram delivery received within ≤ 5min of alert firing
- [ ] Alert resolves automatically once restart-loop subsides (`send_resolved: true` verified)
- [ ] Note: runbook's current Drill 1 targets `client-api` (and "repeat for each of the 6 services"); under THIS story's AC3 the operator targets prometheus + cadvisor + grafana specifically because they are observability containers — failure here is itself an observability blind spot. Runbook completion under AC6 must add this variant.
- [ ] Drill outcome row appended to §Drill Results table in `runbooks/chaos-drill-single-host.md`
- [ ] Post-mortem committed at `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-container-restart-loop.md`

### AC4 — Network partition drill (block egress to SirmaAI)

- [ ] iptables / nftables rule applied on www1 blocking egress to `agenticsai.endigitalx.com` (resolve to current IP; apply both A and AAAA records if dual-stack)
- [ ] `sirmaai-gateway` circuit-breaker (E04 amendment S04.27 / E28 S28.07) observed to transition `closed → open` within ≤ 5min of partition
- [ ] `platform.degraded_mode` event published to Redis Streams (verified via `XINFO STREAMS notification.degraded_mode_events` or equivalent consumer-side log)
- [ ] Tenant-visible banner rendered in frontend Trust Center / system-status surface within 1min of degraded_mode event (per E28 S28.07 hysteresis)
- [ ] Circuit-breaker recovery on partition removal: banner clears within 2min hysteresis window
- [ ] **Dependency note:** AC4 cannot execute until E28 S28.07 (tenant-visible degraded-mode banner) lands. If AC4 needs to execute BEFORE E28, the banner-verification subordinate may be deferred; circuit-breaker open + log evidence alone satisfies the minimum bar.
- [ ] Drill outcome row appended to §Drill Results table in `runbooks/chaos-drill-single-host.md`
- [ ] Post-mortem committed at `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-network-partition.md`

### AC5 — Each drill produces a populated post-mortem

- [ ] Four post-mortems committed under `eusolicit-docs/post-mortems/` (filename pattern `2026-MM-DD-pe-04-<drill-name>.md`)
- [ ] Each post-mortem documents: drill date + operator + start/end times, observed recovery time, alert-routing evidence (email/Telegram/Slack screenshots or log extracts), root-cause analysis if any drill failed or deviated, follow-up tickets opened (if any), re-drill date (if needed)
- [ ] Existing `post-mortems/2026-MM-DD-pe-04-chaos-drill.md` template placeholder either deleted (if all 4 drills have their own post-mortems) or repurposed as an index/summary doc

### AC6 — `runbooks/chaos-drill-single-host.md` §Drill Results table populated + runbook content updated

- [ ] §Drill Results table at `runbooks/chaos-drill-single-host.md:154-161` filled in for all 4 drill rows (Drill 1-4): `Date / Drill / Outcome / Recovery time / Issues found`
- [ ] Runbook content reconciled with the proposal's 4-AC scope:
  - Drill 2 (disk fill) — KEEP as supplementary OR ARCHIVE (operator decides at runbook-completion time); disk-fill alerting verified separately by onprem-04 host-alerts.yaml
  - Drill 1 (container kill) — ADD prometheus + cadvisor + grafana targeting variant per AC3
  - ADD Drill 5 — network-partition-against-SirmaAI procedure per AC4 (iptables rules, observability hooks, partition removal sequence)
- [ ] Closing-line in the runbook updated to read "After all four AC-mandated drills pass, sprint-status row pe-04-chaos-drill-execution can flip to done" (replaces existing "rescoped" wording)
- [ ] Cross-reference to this story spec at `eusolicit-docs/implementation-artifacts/pe-04-chaos-drill-execution.md` added in the runbook's §References

### AC7 — Alertmanager routing verified end-to-end via the drills

- [ ] AC3 (container restart-loop) confirms `severity: page` routing reaches both email + Telegram (validates pe-06 §AC3 page-email-and-telegram receiver in live conditions)
- [ ] AC4 (network partition) confirms degraded-mode banner cascade reaches the tenant UI
- [ ] AC1 + AC2 confirm `severity: ticket` routing reaches Slack (validates pe-06 §AC4 slack-platform-alerts receiver in live conditions) if any alerts fire during postgres/redis recovery (e.g. `PostgresWalArchivingStalled` would fire if WAL archive paused longer than expected during drill)
- [ ] Per-drill alert-routing observation noted in the relevant post-mortem (see AC5)
- [ ] No false negatives observed: every alert rule that SHOULD have fired during a drill DID fire

---

## Dev Notes

### What's on disk (auditable artefacts — INPUTS, not deliverables)

1. **`eusolicit-docs/runbooks/chaos-drill-single-host.md`** (180 lines) — covers 4 drills (container kill, disk fill, postgres crash, redis AOF). Drill 4 (redis AOF) IS specified despite earlier sprint-status row comment claiming "redis drill missing" (audit comment was stale; runbook was updated between 2026-05-11 partial PM dev pass and 2026-05-13 audit). §Drill Results table at lines 154-161 has 4 empty rows awaiting drill execution. **Note:** runbook drill set does NOT match this story's AC set 1:1 — see "Runbook-vs-AC delta" below.
2. **`eusolicit-docs/implementation-artifacts/.archive/pe-04-chaos-drill-runbook.md`** — pre-pivot K8s-flavoured variant; SUPERSEDED-banner already applied. Reference only.
3. **`eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md`** — placeholder template; pattern for the 4 per-drill post-mortems under AC5.

### Runbook-vs-AC delta (critical reconciliation under AC6)

The runbook's existing 4 drills and this story's 4 AC drills are NOT a 1:1 match:

| Runbook drill | Story AC | Reconciliation |
|---|---|---|
| Drill 1: Container kill (single-svc) | — | Repurpose: AC3 variant targets prometheus + cadvisor + grafana; runbook addition needed |
| Drill 2: Disk fill | — | Keep as supplementary OR archive; not in this story's AC scope (covered separately by onprem-04 host-alerts.yaml) |
| Drill 3: Postgres crash | AC1 | 1:1 match |
| Drill 4: Redis AOF replay | AC2 | 1:1 match |
| (none in runbook) | AC4 | NEW: network partition drill — must be authored under AC6 |

The dev agent's AC6 work includes either authoring the new procedures inline in the runbook OR updating runbook to point at sibling sub-procedure files. PM recommendation: keep all 4 drills inline in the single runbook for operator simplicity.

### Carry-forward inputs (proposal §3.1.2 + §4.2)

- `eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md` §3.1.2 + §4.2 — this story's parent disposition
- `eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-12.md` — original epic-23 injection that created the orphan row
- `eusolicit-docs/implementation-artifacts/epic-21-retro-2026-05-05.md` — original carry-forward origin (E21 retro flagged pe-04 as operator-deferred)
- `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md` — ADR-010 establishes the "rehearsed recovery" compensating-control posture this story validates
- `eusolicit-docs/implementation-artifacts/epic-22-retro-2026-05-12.md` § E22-AP01 + E22-AP02 — operator-execution gates do NOT block code-eligible AC close (relevant for §Closure path semantics below)

### Salvaged patterns

- **`docker kill -s KILL <container>` + auto-restart verification loop** — pattern used in Drill 1, 3, 4 in the existing runbook
- **Sentinel row / sentinel key pre-drill** — pattern used in Drill 3 (audit_log row) + Drill 4 (redis SET); same approach for AC1/AC2
- **`/healthz` poll across all 6 service ports** — service-reconnect verification pattern; same for AC1/AC2/AC3
- **Post-mortem template** — already exists at `post-mortems/2026-MM-DD-pe-04-chaos-drill.md`; rename/copy per drill name under AC5

### Anti-pattern guards

- **DO NOT execute drills on production www1 during business hours** — schedule at low-traffic window; announce in `#platform-alerts` ≥30min before; cancel if any tenant-facing degraded-mode banner is already live (would compound user-visible disruption)
- **DO NOT skip cleanup steps** — Drill 2's disk-fill cleanup `sudo rm /tmp/chaos-disk-fill.bin` is non-negotiable (left in place = real on-disk space depletion); same discipline for any new procedures under AC6
- **DO NOT short-circuit AC8 from sibling story pe-06** — pe-06 closed today on the strength of code-eligible AC1-AC7; pe-06's AC8 (operator drill) effectively rolls into THIS story's AC7 alertmanager-routing verification. Don't claim pe-06's AC8 done via this story's drills unless they specifically exercise email + Telegram + Slack paths
- **DO NOT execute AC4 (network partition) before E28 S28.07 lands** — see AC4 dependency note. If E28 timeline slips, escalate to PM for AC4 deferral via dedicated follow-up story rather than weakening the AC bar

### Test approach

This is an **operator-execution** story — there is no automated test surface. Verification = drill execution outcomes documented in post-mortems + runbook §Drill Results table.

Where automated regression IS desirable:
- The `ContainerRestartLoop` alert rule SHOULD have a unit test asserting its PromQL evaluates correctly under synthetic restart-count input. This is **out of scope** for pe-04 (belongs to onprem-04 testing track); if missing, log to deferred-work.md during AC6 runbook completion.
- The `platform.degraded_mode` event consumer (E28 S28.07) SHOULD have an integration test; out of scope for pe-04, owned by E28.

---

## Closure path

1. **Operator executes all 4 drills** — schedule across one or more low-traffic windows within the quarter; PM recommendation is to batch AC1 + AC2 in the same maintenance window (postgres + redis crashes can be sequenced 15min apart), and run AC3 + AC4 separately
2. **AC1-AC4 outcomes recorded** in per-drill post-mortems per AC5
3. **AC6 runbook updates** completed as drills run (incremental — add procedures + fill §Drill Results rows as each drill executes)
4. **AC7 routing observations** noted in each post-mortem
5. **PM (📋 John) reviews** the 4 post-mortems + the populated runbook §Drill Results table for completeness
6. **bmad-code-review** dispatched against the story spec (this file) + the 4 post-mortems + the updated runbook (audit-style review, similar to pe-06 backfill pattern)
7. **On Approve verdict** — AP17-C1 two-gate close: story file `Status: review → done` + sprint-status `pe-04-chaos-drill-execution: ready-for-dev → done` (per AP18-C2 atomic) in same commit-equivalent
8. **Epic 23 close** — pe-04 done is one of 4 remaining gates (alongside drift-recovery code-Approve, public-sla normal dispatch, inj-03 TEA escalation)

---

## Risk acknowledgment

Per proposal §3.3:

- **Operator-paced execution** — story may sit at `in-progress` across multiple sprint windows (quarterly drill cadence per runbook). Mitigation: optional decomposition into 4 sibling stories (`pe-04-chaos-drill-postgres`, `pe-04-chaos-drill-redis`, `pe-04-chaos-drill-container-restart-loop`, `pe-04-chaos-drill-network-partition`) at operator preference. Trigger: if first drill window doesn't accommodate all 4 ACs.
- **AC4 (network partition) blocks on E28 S28.07** — see AC4 dependency note. PM-owned coordination via SirmaAI pivot sprint planning.
- **AC3 / AC7 alert-routing failure** — if a drill exposes a regression in pe-06 routing (closed today 2026-05-13), open a defect ticket and trigger pe-06 REVIEW-FIX scope tied to the failing AC. Do NOT silently absorb; the routing closure was an audit-backfill on landed code, so drill exposure of a routing gap IS the AC8 operator-drill outcome that pe-06 explicitly deferred.

---

## See also

- `eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md` § 3.1.2 + § 4.2 — parent disposition + this story's spec sketch
- `eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-12.md` — original epic-23 injection
- `eusolicit-docs/runbooks/chaos-drill-single-host.md` — drill procedures (the operator's working document)
- `eusolicit-docs/implementation-artifacts/pe-06-pagerduty-rotation-provisioning.md` — sibling story; alertmanager routing AC7 validates against this
- `eusolicit-docs/implementation-artifacts/epic-22-retro-2026-05-12.md` § E22-AP02 — operator-execution gate semantics
- `eusolicit-docs/planning-artifacts/epics/E23-operational-close-out.md` — owning epic
