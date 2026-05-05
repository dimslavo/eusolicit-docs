# Post-Mortem: PE.04 Chaos Drill (Node Drain Exercise)

**Version**: SKELETON — operator overwrites date + populates content post-drill
**Story**: PE.06 (21-6) — AC-4 (first chaos-drill post-mortem; D-3 operator-deferred)
**Last updated**: 2026-05-05

> **D-3 pre-recorded deviation**: Live drill execution + post-mortem completion is operator-action.
> Autopilot ships this skeleton with operator-capture placeholders. The operator runs the drill
> against the PE.06 runbooks, updates this file in-place, and commits the populated post-mortem.
>
> **First entry**: This is the FIRST file in the `eusolicit-docs/post-mortems/` directory,
> establishing the post-mortem repository. Future post-mortems append as new files in this directory.
>
> **Rename instruction**: Replace `MM-DD` in the filename with the actual drill date before committing.
> e.g., `2026-05-14-pe-04-chaos-drill.md`

---

```yaml
# ── Frontmatter ─────────────────────────────────────────────────────────────
incident_id: "pe-04-chaos-drill-2026-MM-DD"
severity: "SEV-2 (synthetic — staging only)"
start_time: "<YYYY-MM-DDTHH:MM:SSZ>"       # Drill start time
detect_time: "<YYYY-MM-DDTHH:MM:SSZ>"      # Same as start (drill is deliberate)
ack_time: "<YYYY-MM-DDTHH:MM:SSZ>"         # Drill IC acknowledged drill start
resolve_time: "<YYYY-MM-DDTHH:MM:SSZ>"     # All drain/failover cycles complete
impact_duration: "<X minutes>"             # Duration of drill
customers_impacted: "0 (staging environment only)"
error_budget_consumed_pct: "0% (synthetic drill; staging SLO not tracked)"
runbooks_followed:
  - "node-drain.md"
  - "redis-failover.md (if Redis pod drained)"
  - "pg-failover.md (N/A — managed RDS not drained; AWS Multi-AZ failover is separate)"
ic: "<PLACEHOLDER: Incident Commander for the drill>"
drill_rota:
  # List each service drained during the drill (per pe-04-chaos-drill-runbook.md §Per-Service Drill)
  - service: "client-api"
    node_drained: "<node-name>"
    duration_seconds: "<X>"
    replicas_min_maintained: "<true|false>"
    pdb_blocked_eviction: "<true|false>"
    reconnect_sla_met: "N/A (application-layer service; no DB reconnect)"
  - service: "admin-api"
    node_drained: "<node-name>"
    duration_seconds: "<X>"
    replicas_min_maintained: "<true|false>"
    pdb_blocked_eviction: "<true|false>"
    reconnect_sla_met: "N/A"
  - service: "ai-gateway"
    node_drained: "<node-name>"
    duration_seconds: "<X>"
    replicas_min_maintained: "<true|false>"
    pdb_blocked_eviction: "<true|false>"
    reconnect_sla_met: "N/A"
  - service: "data-pipeline"
    node_drained: "<node-name>"
    duration_seconds: "<X>"
    replicas_min_maintained: "<true|false>"
    pdb_blocked_eviction: "<true|false>"
    reconnect_sla_met: "N/A"
  - service: "notification"
    node_drained: "<node-name>"
    duration_seconds: "<X>"
    replicas_min_maintained: "<true|false>"
    pdb_blocked_eviction: "<true|false>"
    reconnect_sla_met: "N/A"
  - service: "integrations-api"
    node_drained: "<node-name>"
    duration_seconds: "<X>"
    replicas_min_maintained: "<true|false>"
    pdb_blocked_eviction: "<true|false>"
    reconnect_sla_met: "N/A"
runbooks_gaps_found:
  # Operator populates after drill: list any runbook step that was unclear, missing, or wrong
  - runbook: "<runbook-id>.md"
    section: "<§Triage|§Resolution|§Verification>"
    gap: "<description of the gap>"
    action: "<proposed fix to the runbook>"
```

---

## Executive Summary

> **Operator placeholder — replace with actual post-drill summary.**

**Paragraph 1**: On `<DATE>` at `<TIME>` UTC, the EU Solicit platform team executed the PE.04 chaos drill per `pe-04-chaos-drill-runbook.md` §Per-Service Drill in the staging environment. The drill validated PDB + min-replica behaviour across all 6 services by draining nodes sequentially and observing pod eviction, rescheduling, and reconnect behaviour. This drill serves as the first incident-response exercise per epic line 153 and is the first post-mortem in the `eusolicit-docs/post-mortems/` repository.

**Paragraph 2**: `<OPERATOR: Summarise the outcome. Did all services maintain minAvailable? Did PDB work correctly? Were runbook gaps found? What action items resulted? Example: "All 6 services maintained ≥1 ready replica throughout the drill, confirming PDB + min-replica invariants from Story 21-4. Two runbook gaps were identified in node-drain.md (missing HPA scale-out verification step) and redis-failover.md (Lua-script re-run timing was unclear). Both gaps are logged as action items below.">`

---

## Timeline

> **NO "who" column. Columns: Time (UTC) | What happened | Why it happened.**
> Operator populates this after the drill.

| Time (UTC) | What happened | Why it happened |
|------------|---------------|----------------|
| `<HH:MM>` | Drill announced in Slack `#platform-incidents` | Planned exercise per PE.04 chaos-drill runbook |
| `<HH:MM>` | Staging cluster confirmed at nominal state (all 6 services ≥2 replicas) | Pre-drill checklist per `node-drain.md` §Triage |
| `<HH:MM>` | Node `<node-name>` cordoned | First drain initiated (client-api) |
| `<HH:MM>` | `kubectl drain <node-name>` executed | Operator-initiated per `node-drain.md` §Resolution Branch A |
| `<HH:MM>` | PDB triggered — eviction blocked for `<service>` | Correct PDB behaviour; `ALLOWED DISRUPTIONS = 0` at this moment |
| `<HH:MM>` | Pod rescheduled on new node; service returns to minReplicas | `<PLACEHOLDER>` |
| `<HH:MM>` | `<Next service drain step>` | `<PLACEHOLDER>` |
| `<HH:MM>` | All 6 service drains complete | Drill cycle complete |
| `<HH:MM>` | Verification checks completed | All services at minReplicas; no customer-facing impact |
| `<HH:MM>` | Drill closed | `<PLACEHOLDER>` |

---

## Root Cause

> **Not applicable for a synthetic drill.** The drill is deliberate, not a failure.

Root cause: N/A — synthetic drill. Node drains were operator-initiated per `pe-04-chaos-drill-runbook.md` §Per-Service Drill to validate PDB and min-replica invariants in staging.

5-Whys: N/A for synthetic drill. Any runbook gaps found are tracked in §What Went Poorly and §Action Items.

---

## Contributing Factors

> Operator populates: what made the drill harder or easier than expected.

| Factor | Category | Mitigation |
|--------|----------|-----------|
| `<PLACEHOLDER: e.g., "HPA scale-out was slower than expected (90s vs. expected 60s) because the EKS node group needed time to provision a new node">` | Tooling gap | `<PLACEHOLDER>` |
| `<PLACEHOLDER>` | `<Process gap / Tooling gap / Knowledge gap>` | `<PLACEHOLDER>` |

---

## What Went Well

> Operator populates after drill.

- `<PLACEHOLDER: e.g., "PDB enforcement worked correctly — no service dropped below 1 ready replica during any drain.">`
- `<PLACEHOLDER: e.g., "The node-drain.md runbook's pre-drain checklist prevented a premature drain attempt when one service was at minReplicas.">`
- `<PLACEHOLDER: e.g., "PagerDuty's runbook_url annotation in the alert was tested and confirmed functional during the drill.">`

---

## What Went Poorly

> Operator populates after drill. System/process gaps only — no individual names.

- `<PLACEHOLDER: e.g., "The node-drain.md runbook did not include a step for verifying HPA scale-out after drain — added to action items below.">`
- `<PLACEHOLDER: e.g., "The redis-failover.md runbook's Lua-script re-run timing was unclear — when exactly should the operator check for NOSCRIPT errors?">`

---

## Action Items

> Operator populates after drill. Each runbook gap becomes an action item.
> Sprint-trackable items should be injected as new stories.

| Action | Owner | Due | Status | Story |
|--------|-------|-----|--------|-------|
| `<PLACEHOLDER: e.g., "Add HPA scale-out verification step to node-drain.md §Verification">` | Platform team | `<YYYY-MM-DD>` | Open | TBD |
| `<PLACEHOLDER>` | Platform team | `<YYYY-MM-DD>` | Open | TBD |

---

## Lessons Learned

> Operator writes 1–3 paragraphs after drill. What the drill taught about the system, process, or tooling.

**Paragraph 1**: `<PLACEHOLDER: What the drill taught about PDB + min-replica behaviour in production-like conditions.>`

**Paragraph 2**: `<PLACEHOLDER: What the drill taught about the runbook quality (node-drain.md, redis-failover.md, pg-failover.md). Which steps were clear? Which were ambiguous?>`

**Paragraph 3** (optional): `<PLACEHOLDER: Positive reinforcement. What confirmed design decisions? What should be preserved?>`

---

## Sign-off

| Role | Name | Date |
|------|------|------|
| Incident Commander (IC) | `<PLACEHOLDER>` | `<YYYY-MM-DD>` |
| Drill Reviewer | `<PLACEHOLDER>` | `<YYYY-MM-DD>` |
| CTO | `<PLACEHOLDER>` | `<YYYY-MM-DD>` |

> **2-week soak gate note**: Once this drill is signed off and the PagerDuty rotation has been
> active for ≥2 weeks with at least one real page (or the Story 21-5 AC-9 synthetic burn-rate
> test page), the operator updates `pe-06-incident-readiness-runbook.md` §Sign-off to record
> the soak completion date. The public 99.9% SLA announcement is unblocked at that point.

*Post-mortem approved and committed to `eusolicit-docs/post-mortems/` as the first entry.*
