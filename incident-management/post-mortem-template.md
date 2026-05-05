# Post-Mortem Template — EU Solicit

**Version**: 1.0 | **Story**: PE.06 (21-6) | **Last updated**: 2026-05-05

> **Blameless culture**: This template is structurally blameless. There is NO "who" column in
> the Timeline section — only "what" and "why". The Action Items table has an "Owner" column
> for remediation accountability, not for blame. Systems and processes fail; individuals respond.
>
> Structural exemplar: Epic 13 retrospective at
> `eusolicit-docs/implementation-artifacts/epic-13-retro-2026-04-26.md`.
>
> Reference: Google SRE Workbook §16 (post-mortem culture).

---

## Instructions for use

1. Copy this file to `eusolicit-docs/post-mortems/<YYYY-MM-DD>-<incident-slug>.md`.
2. Fill in the frontmatter and all sections.
3. Replace all `<PLACEHOLDER>` markers with real content.
4. Review with IC + all engineers involved before publishing.
5. **NEVER attribute fault to individuals.** If tempted to write a person's name in the Timeline or Root Cause, write the system/process/tool name instead.
6. Schedule the post-mortem meeting within 5 business days for SEV-1/2.

---

```yaml
# ── Frontmatter ─────────────────────────────────────────────────────────────
incident_id: "<unique-id>"          # e.g., "pe-04-chaos-drill-2026-05-10" or "inc-2026-06-15-db-failover"
severity: "SEV-<1|2|3>"
start_time: "<YYYY-MM-DDTHH:MM:SSZ>"   # When the condition began (not when it was detected)
detect_time: "<YYYY-MM-DDTHH:MM:SSZ>"  # When alerting or customer report triggered response
ack_time: "<YYYY-MM-DDTHH:MM:SSZ>"     # When on-call acknowledged
resolve_time: "<YYYY-MM-DDTHH:MM:SSZ>" # When verified resolution complete
impact_duration: "<X minutes / hours>"  # resolve_time - start_time
customers_impacted: "<N tenants / 0 (staging) / unknown>"
error_budget_consumed_pct: "<X%>"       # Percentage of monthly error budget consumed
runbooks_followed:
  - "<runbook-id>.md"                   # e.g., "pg-failover.md"
  - "<runbook-id>.md"
ic: "<Incident Commander name>"         # Do NOT add blame context; this is IC identification only
```

---

## Executive Summary

> **2 paragraphs. What happened, why it matters, what was fixed.**

**Paragraph 1** — What happened: `<PLACEHOLDER: Brief factual description of the incident — what failed, when, for how long, who was affected. No blame. Example: "On 2026-MM-DD at HH:MM UTC, the PostgreSQL primary node experienced a disk-full condition that blocked all write operations for X minutes. Approximately N customers experienced proposal submission failures during this window.">`

**Paragraph 2** — Outcome: `<PLACEHOLDER: What was done to resolve it, what was learned, and what the follow-up action items are. Example: "The on-call engineer identified and resolved the disk pressure within X minutes by triggering RDS storage auto-scaling. A post-mortem identified a missing autovacuum configuration and an absent disk-utilisation alert as contributing factors. Both are tracked as action items below.">`

---

## Timeline

> **Format**: chronological. Columns: **Time** (UTC) | **What happened** | **Why it happened** (or "unknown at the time").
> **NO "who" column** — this is a structural blameless invariant. The "what" and "why" columns
> carry all information needed for root-cause analysis. Individual names may appear in the IC
> or Slack attribution but are NEVER in the Timeline to prevent retroactive blame-finding.

| Time (UTC) | What happened | Why it happened |
|------------|---------------|----------------|
| `<HH:MM>` | `<PLACEHOLDER: Incident began — describe the observable event>` | `<PLACEHOLDER: Root cause known at the time, or "Not known at this point">` |
| `<HH:MM>` | `<PLACEHOLDER: First alert fired or customer report received>` | `<PLACEHOLDER: Alert threshold exceeded>` |
| `<HH:MM>` | `<PLACEHOLDER: On-call acknowledged PagerDuty page>` | — |
| `<HH:MM>` | `<PLACEHOLDER: IC declared; runbook opened>` | — |
| `<HH:MM>` | `<PLACEHOLDER: First resolution action taken>` | `<PLACEHOLDER: Why this action was chosen>` |
| `<HH:MM>` | `<PLACEHOLDER: Resolution action outcome>` | — |
| `<HH:MM>` | `<PLACEHOLDER: Verification checks passed>` | — |
| `<HH:MM>` | `<PLACEHOLDER: Incident resolved in PagerDuty>` | — |

---

## Root Cause

> **Single sentence** + supporting evidence. Use the **5-Whys** technique.
> Root cause is always a **system or process gap**, never an individual action.

**Root cause**: `<PLACEHOLDER: One sentence. Example: "RDS storage auto-scaling was enabled but the minimum free-space trigger threshold was set to 10% of a value that had not been updated after storage grew, causing the auto-scale to fail silently until storage was completely exhausted.">`

**5-Whys analysis**:
1. Why? `<PLACEHOLDER>`
2. Why? `<PLACEHOLDER>`
3. Why? `<PLACEHOLDER>`
4. Why? `<PLACEHOLDER>`
5. Why? `<PLACEHOLDER: Root cause is the answer to the final "why">`

---

## Contributing Factors

> Not the root cause, but conditions that made the incident worse or harder to detect.
> These are **process gaps**, **tooling gaps**, or **knowledge gaps** — not individual actions.

| Factor | Category | Mitigation |
|--------|----------|-----------|
| `<PLACEHOLDER: e.g., "No disk-utilisation alert at 80% threshold">` | Tooling gap | `<PLACEHOLDER: e.g., "Add CloudWatch alarm for FreeStorageSpace < 10GB">` |
| `<PLACEHOLDER>` | `<Process gap / Tooling gap / Knowledge gap>` | `<PLACEHOLDER>` |
| `<PLACEHOLDER>` | `<Process gap / Tooling gap / Knowledge gap>` | `<PLACEHOLDER>` |

---

## What Went Well

> Positive observations — what the system, process, or tooling did correctly.
> This section prevents the post-mortem from becoming purely negative and identifies
> what should be preserved and reinforced.

- `<PLACEHOLDER: e.g., "The PagerDuty alert fired within 30 seconds of the condition exceeding threshold.">`
- `<PLACEHOLDER: e.g., "The pg-failover.md runbook clearly described the verification steps; the IC followed them without deviation.">`
- `<PLACEHOLDER: e.g., "The on-call rotation escalation to backup worked correctly when primary did not respond within 5 minutes.">`

---

## What Went Poorly

> Negative observations — where systems, processes, or tooling fell short.
> Focus on the system, not the action. "The runbook lacked a step for X" is correct.
> "Someone forgot to do X" is not appropriate here.

- `<PLACEHOLDER: e.g., "The full-disk-on-pg.md runbook did not include a step to verify storage auto-scaling configuration before declaring the incident resolved.">`
- `<PLACEHOLDER: e.g., "The Alertmanager alert for FreeStorageSpace was missing entirely — detection was via customer report, not automated alerting.">`
- `<PLACEHOLDER>`

---

## Action Items

> Sprint-trackable items. **Owner** is the assigned engineer for remediation — NOT a blame target.
> Items should be specific, time-bound, and verifiable.

| Action | Owner | Due | Status | Story |
|--------|-------|-----|--------|-------|
| `<PLACEHOLDER: e.g., "Add CloudWatch alarm for RDS FreeStorageSpace < 5GB + wire to #platform-alerts">` | `<engineer or team>` | `<YYYY-MM-DD>` | `Open` | `<story key or TBD>` |
| `<PLACEHOLDER>` | `<engineer or team>` | `<YYYY-MM-DD>` | `Open` | `<story key or TBD>` |
| `<PLACEHOLDER>` | `<engineer or team>` | `<YYYY-MM-DD>` | `Open` | `<story key or TBD>` |

> **Sprint injection**: Action items with sprint-trackable scope should be created as new stories via
> the PB-STORY-003 mid-sprint story creation playbook. Reference the story key here once created.

---

## Lessons Learned

> **1–3 paragraphs.** Synthesis — what this incident taught the team about the system,
> the process, or the tooling. Written for an engineer joining the team in 6 months.
> This section is the lasting value of the post-mortem.

**Paragraph 1**: `<PLACEHOLDER: System/architecture lesson. Example: "This incident revealed that RDS storage auto-scaling, while enabled, has a non-obvious trigger threshold that can be misconfigured without any visible error. The safest pattern for RDS storage is to set both MaxAllocatedStorage AND a CloudWatch alarm at 10GB remaining, so the alarm fires before auto-scaling is needed.">`

**Paragraph 2**: `<PLACEHOLDER: Process lesson. Example: "The 5-minute ack SLA for SEV-1 incidents was met in this incident (ack within 3 minutes), but the runbook's triage section lacked enough specificity to avoid a false-positive resolution that required re-opening the incident 20 minutes later. Runbooks should include a 'false-resolution guard' verification step that re-checks the metric 5 minutes after fix is applied.">`

**Paragraph 3** (optional): `<PLACEHOLDER: Tooling lesson or positive reinforcement. Example: "The PagerDuty runbook_url annotation in the alert proved valuable — the IC opened the correct runbook within 90 seconds of the page. This confirms the PE.05 + PE.06 design decision to include runbook_url in every alerting rule was correct.">`

---

## Sign-off

| Role | Name | Date |
|------|------|------|
| Incident Commander | `<PLACEHOLDER>` | `<YYYY-MM-DD>` |
| Reviewer | `<PLACEHOLDER>` | `<YYYY-MM-DD>` |
| CTO (SEV-1 only) | `<PLACEHOLDER>` | `<YYYY-MM-DD>` |

*Post-mortem approved for publication.*
