# EU Solicit — Incident Response Process

**Version**: 1.0 | **Story**: PE.06 (21-6) | **Last updated**: 2026-05-05

> **IC-led process**: Every declared incident has a single Incident Commander (IC).
> The IC is the decision-maker and communicator. All engineers involved take
> direction from the IC. This prevents parallel actions that contradict each other
> and keeps the incident timeline coherent.

---

## Process Overview

```
Detection → Ack → IC Declared → Comms Channel → Diagnose → Fix → Verify → Resolve → Post-Mortem
    1          2        3              4              5         6       7        8           9
```

---

## Step 1 — Detection

**How an incident is detected**:

| Source | Action |
|--------|--------|
| **PagerDuty page** (`severity=page` alert from Alertmanager) | IC is paged. See `severity-definitions.md` §SEV-1. |
| **Slack `#platform-alerts` notification** (`severity=ticket` alert) | On-call engineer reviews. See `severity-definitions.md` §SEV-2. |
| **Proactive on-call observation** (Grafana dashboard anomaly, customer spike) | On-call engineer self-initiates. Declare severity per `severity-definitions.md`. |
| **Customer report** (support channel, email, direct message) | Triage engineer assesses. Declare severity. |

**Action**: Note the detection time. This is the SLA clock start per `severity-definitions.md` §Response-Time Clock Start.

---

## Step 2 — Acknowledge

**SLA targets per `severity-definitions.md`**:
- SEV-1: ack PagerDuty within **5 minutes**.
- SEV-2: ack Slack notification within **15 minutes**.
- SEV-3: next business day review.

**How to ack**:
- PagerDuty: press "Acknowledge" in the PagerDuty mobile app or web console.
- Slack: post a reply in `#platform-incidents` (even just "Acknowledged — investigating"):
  ```
  :eyes: [<time>] Acknowledged. On-call: <name>. Investigating <brief description>.
  ```
- If on-call primary does not ack within 5 minutes → PagerDuty escalation policy fires to backup.

---

## Step 3 — Incident Commander (IC) Declared

**Who is the IC**:
- **Default**: on-call primary.
- **If primary unavailable**: on-call backup.
- **If SEV-1 with data-integrity or security scope**: CTO joins as observer. IC remains on-call primary unless CTO explicitly takes IC role.

**How to declare**:
Post in Slack `#platform-incidents`:
```
:fire: [<time>] INCIDENT DECLARED — <SEV-1|SEV-2>
IC: <name>
Summary: <one-sentence description of impact>
Runbook being followed: <runbook URL or "TBD">
```

**IC responsibilities**:
1. Own the incident timeline (what was tried, what worked, what failed).
2. Coordinate diagnosis (direct other engineers to specific runbook steps).
3. Gate all resolution actions (one engineer acts at a time; parallel actions = chaos).
4. Drive customer communication per `status-page-comms-templates.md`.
5. Hand off to backup IC if primary needs sleep (SEV-1 overnight).

---

## Step 4 — Comms Channel Opened + Status Page Updated

**Slack `#platform-incidents`**:
- The Story 16.0 integrations-api Slack webhook enables posting to `#platform-incidents`.
- **No new Slack integration needed** — use the existing webhook (anti-pattern guard #7).
- Format: structured updates every 30 minutes for SEV-1, every 1 hour for SEV-2.

**Status page** (for SEV-1 customer-visible incidents):
- Copy-paste from `status-page-comms-templates.md` SEV-1 Initial template.
- Post manually to the platform status channel (Statuspage.io integration deferred to future epic — anti-pattern guard #7).
- Update every 1 hour until resolved (`status-page-comms-templates.md` SEV-1 Update).

**Slack `#platform-incidents` update format**:
```
:mega: [<time>] UPDATE — SEV-<N> <incident_id>
Status: <Investigating|Identified|Fixing|Monitoring>
IC: <name>
Current understanding: <hypothesis or confirmed root cause>
Action taken: <what was done in last 30 min>
Next update: <time>
```

---

## Step 5 — Diagnose Using Runbooks

**PagerDuty alert payload includes `runbook_url`** per PE.05 AC-7.3 contract:
- Every alert from `alerting-rules.yaml` and `kraftdata-isolation-rules.yaml` includes a `runbook_url` annotation.
- PagerDuty displays the URL in the alert details.
- Open the runbook URL → follow §Triage → follow §Resolution.

**Runbook URL convention**: `https://github.com/eusolicit/eusolicit/blob/main/eusolicit-docs/runbooks/<runbook-id>.md`

**If no runbook URL** (customer-reported incident, no alert fired):
1. Identify the affected service.
2. Check per-service Grafana dashboard.
3. Use the most relevant runbook from `eusolicit-docs/runbooks/` based on observed symptoms.

**Diagnosis principle**: NEVER assume. Always triage to confirm the diagnosis before taking resolution action.

---

## Step 6 — Fix Per Runbook §Resolution

1. IC announces the planned fix in `#platform-incidents`:
   ```
   :wrench: [<time>] EXECUTING FIX
   Action: <one-sentence description, e.g., "helm rollback client-api to revision 14">
   Expected outcome: <one-sentence, e.g., "error rate returns to baseline within 5 minutes">
   ```
2. One engineer executes the fix. Others observe and report.
3. If fix is disruptive (e.g., a failover), announce it before executing.
4. If fix is the wrong call (worsens the situation) → IC calls HALT → follow runbook §Rollback.

---

## Step 7 — Verify Per Runbook §Verification

1. Run every verification check in the runbook §Verification section.
2. Record results in `#platform-incidents`.
3. Do NOT declare the incident resolved until ALL verification checks pass.

**Minimum verification criteria** (in addition to runbook-specific checks):
- Error rate returned to baseline (Grafana / Prometheus).
- p95 latency returned to baseline.
- PagerDuty alert auto-resolves (or manually resolved after verification).

---

## Step 8 — Resolve in PagerDuty + Slack

**PagerDuty**:
- Press "Resolve" in PagerDuty once all verification checks pass.
- Add a brief resolution note: `"<root cause one-sentence>. Fixed by <runbook §Resolution branch>. Verified via <verification step>."`

**Slack `#platform-incidents`**:
```
:white_check_mark: [<time>] INCIDENT RESOLVED — SEV-<N> <incident_id>
Duration: <start_time> to <end_time> (<X minutes>)
Root cause: <one sentence>
Fix applied: <brief description>
Post-mortem: <"Scheduled within 5 business days" | "Optional (SEV-3)">
```

**Status page** (if SEV-1 customer-visible):
- Copy-paste from `status-page-comms-templates.md` SEV-1 Resolved template.

---

## Step 9 — Schedule Post-Mortem

| Severity | Post-mortem requirement |
|----------|------------------------|
| SEV-1 | **Mandatory** — schedule within 5 business days |
| SEV-2 | **Required** if incident lasted >30 min or recurred within 30 days |
| SEV-3 | Optional |

**Post-mortem format**: use `post-mortem-template.md`.
**Post-mortem location**: `eusolicit-docs/post-mortems/<YYYY-MM-DD>-<incident-slug>.md`.
**Post-mortem meeting**: invite IC, all engineers involved, and CTO (as observer for SEV-1).
**Blameless rule**: the post-mortem focuses on systems and processes, NOT individuals. See `post-mortem-template.md` §Anti-pattern fence.

**Sprint injection**: if the post-mortem §Action Items table contains sprint-trackable items, inject them as new stories per the PB-STORY-003 mid-sprint story creation playbook.

---

## Escalation Matrix

| Condition | Action |
|-----------|--------|
| On-call primary does not ack in 5 min | PagerDuty escalation policy fires backup |
| On-call backup does not ack in 10 min | PagerDuty escalation policy notifies CTO |
| SEV-1 active for >2 hours with no fix | IC pages CTO directly; options reviewed |
| Data-integrity breach suspected | CTO and engineering lead immediately involved (not optional) |
| Security incident | CTO + legal informed immediately; incident-response channel restricted to need-to-know |

---

## References

- `severity-definitions.md` — SEV-1/2/3 definitions + response SLAs + SLA-scope table
- `post-mortem-template.md` — blameless post-mortem format
- `status-page-comms-templates.md` — SEV-1 status page copy-paste templates
- `eusolicit-docs/runbooks/` — all operational runbooks (12 runbooks; opened via PagerDuty `runbook_url`)
- `incident-management/` — this directory (incident process home)
- PagerDuty escalation policy: `eusolicit-platform-escalation` (PE.06 Terraform module)
- Slack `#platform-incidents` channel — Story 16.0 integrations-api Slack webhook
