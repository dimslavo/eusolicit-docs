# PE.06 Incident Readiness Runbook — On-Call Rotation + Runbooks + Incident Process

**Story:** 21-6-on-call-rotation-runbook-authoring-incident-management-process
**Date Authored:** 2026-05-05
**Epic:** E21 — Platform Reliability for 99.9% SLA
**Status:** DEV-PASS (live PagerDuty rotation + 2-week soak deferred — D-3 operator-action gate)
**Verdict:** DEFERRED — See §Sign-off

---

## §Pre-flight Checklist

The operator completes this checklist before applying PE.06 Terraform resources to staging/production and before activating the on-call rotation.

```
[ ] PagerDuty account exists and PAGERDUTY_TOKEN env var is set in Terraform environment
    — NEVER commit the API token (AP-GUARD-2: same rule as ADR-002)
[ ] Terraform plan reviewed — modules/oncall shows pagerduty_user, pagerduty_schedule,
    pagerduty_escalation_policy, pagerduty_service x2, pagerduty_service_integration x2
[ ] dev/terraform.tfvars has enable_oncall = false (AP-GUARD-1: no PagerDuty on docker-compose loop)
[ ] staging/terraform.tfvars has enable_oncall = true
[ ] prod/terraform.tfvars has enable_oncall = true
[ ] Platform engineer emails confirmed in terraform.tfvars platform_engineers list
[ ] CTO email confirmed in terraform.tfvars cto_email value
[ ] AWS Secrets Manager paths writable by Terraform IAM role:
    — eusolicit/<env>/observability/pagerduty-key
    — eusolicit/<env>/observability/pagerduty-test-key
[ ] PE.05 ESO ExternalSecret confirmed polling eusolicit/<env>/observability/pagerduty-key
    — file: infra/observability/alertmanager/externalsecret.yaml
[ ] Alertmanager receiver block confirmed at infra/observability/alertmanager/alertmanager.yaml lines 70–78
[ ] All ATDD tests GREEN: 15/15 passed
    — run: pytest tests/unit/test_runbook_url_coverage.py -v
[ ] Lint gate passes:
    — run: python3 scripts/check_runbook_url_coverage.py
    — expected: runbook coverage: 7/7 URLs resolve, 5/5 structural checks pass
```

---

## §Implementation Decision

### (a) PagerDuty vs. Opsgenie

**Decision:** PagerDuty chosen as the project default per Epic 21 line 149 ("PagerDuty (or Opsgenie)").

**Rationale:**
- PagerDuty is the most widely adopted on-call management platform with native Prometheus Alertmanager integration (events_api_v2) and a mature Terraform provider (`PagerDuty/pagerduty ~> 3.0`).
- The Alertmanager `receiver: pagerduty` block already ships in `infra/observability/alertmanager/alertmanager.yaml` lines 70–78, establishing PagerDuty as the project default prior to this story.
- For a 2–3 person team, PagerDuty's free tier supports up to 5 users, covering the initial team.

**Opsgenie swap path (operator-elected alternative):**
1. Replace `PagerDuty/pagerduty ~> 3.0` with `opsgenie` provider in `modules/oncall/versions.tf`.
2. Rewrite `modules/oncall/main.tf` using `opsgenie_team`, `opsgenie_schedule`, `opsgenie_escalation_policy`, `opsgenie_integration` resources.
3. Swap `pagerduty` receiver block in `alertmanager.yaml` for `opsgenie_api_key: /etc/alertmanager/secrets/opsgenie-key`.
4. Update ESO ExternalSecret path accordingly.

The cost difference between PagerDuty Professional and Opsgenie Essentials is approximately equivalent at the 3-engineer scale. The decision is a 1-file provider swap + 1-file alertmanager edit.

### (b) Shared Rotation vs. Follow-the-Sun

**Decision:** Single shared rotation per Epic 21 line 149 ("initially shared rotation; can split engineering/platform later").

**Rationale:**
- Follow-the-sun requires ≥6 engineers across ≥3 timezones to be operationally viable without burning out engineers. The current 2–3 person platform team is below this threshold.
- The PagerDuty schedule uses `time_zone = "Europe/Berlin"` (CET/CEST) — all platform engineers are in the EU timezone band.
- Escalation policy: Primary (5-min ack) → Backup (10-min ack) → CTO (30-min ack). The CTO is the final escalation only; they are not in the rotation.
- Rotation can be split engineering/platform once the team grows beyond 4 engineers or once sub-teams emerge (e.g. a dedicated ML/AI team for AI-Gateway incidents vs. a platform team for infra incidents).

### (c) Separate Test Service for Synthetic Burn-Rate Alert

**Decision:** Two PagerDuty services provisioned — `eusolicit-platform-prod` (high-urgency, `severity=page` alerts) and `eusolicit-test-burn-rate-alert` (low-urgency, Story 21-5 AC-9 synthetic e2e test).

**Rationale:**
- Story 21-5 AC-9 fires a synthetic `HighErrorBudgetBurnRate` alert to validate the full alerting path. If this fires into the production on-call service, it pages engineers unnecessarily.
- The low-urgency test service receives the synthetic page (no mobile push, no voice call) — only email/app notification.
- This closes PE.05 D-4 (TEST PagerDuty service operator-action deferred in Story 21-5).

---

## §On-Call Rotation Verification

> **D-3 pre-recorded deviation**: Live PagerDuty rotation + operator capture is an operator action
> that occurs after Terraform apply to staging. The operator populates this section after the rotation
> is live and captures a screenshot of the PagerDuty schedule.

### Terraform Module Resources

| Resource | Count | Purpose |
|----------|-------|---------|
| `pagerduty_user` | N (1 per engineer) | Creates PagerDuty user accounts for each platform engineer |
| `pagerduty_schedule` | 1 | Shared rotation; 2 layers: primary (weekly) + backup (24h offset) |
| `pagerduty_escalation_policy` | 1 | 3-level escalation: primary (5min) → backup (10min) → CTO (30min) |
| `pagerduty_service` | 2 | `eusolicit-platform-prod` (high-urgency) + `eusolicit-test-burn-rate-alert` (low-urgency) |
| `pagerduty_service_integration` | 2 | Prometheus events_api_v2 integration; one per service |
| `aws_secretsmanager_secret` | 2 | Declares secret slots for prod + test integration keys |
| `aws_secretsmanager_secret_version` | 2 | Writes integration keys from Terraform outputs into Secrets Manager |

All resources use `count = var.enabled ? 1 : 0` — zero resources provisioned when `enabled = false` (dev environment).

### PagerDuty Schedule Configuration

```
Schedule: "EU Solicit On-Call"
Timezone: Europe/Berlin
Layer 1 (primary):
  - Rotation type: weekly
  - Start day: Monday 09:00
  - Users: [platform-engineer-1, platform-engineer-2, (platform-engineer-3)]
Layer 2 (backup):
  - Rotation type: weekly
  - Start day: Monday 09:00 + 24h offset
  - Users: [platform-engineer-2, platform-engineer-1, (platform-engineer-3)]
```

### Operator Capture Checklist

```
[ ] PagerDuty schedule screenshot: [OPERATOR CAPTURE — link here after Terraform apply]
    URL: https://app.pagerduty.com/schedules/<schedule-id>
[ ] Escalation policy verified: Primary → Backup → CTO with correct ack timeouts
[ ] PD service "eusolicit-platform-prod" integration key written to:
    AWS Secrets Manager: eusolicit/<env>/observability/pagerduty-key
[ ] PD service "eusolicit-test-burn-rate-alert" integration key written to:
    AWS Secrets Manager: eusolicit/<env>/observability/pagerduty-test-key
[ ] ESO ExternalSecret synced (Alertmanager pod restarts with live key):
    kubectl get externalsecret pagerduty-key -n monitoring
[ ] Test page sent via synthetic burn-rate test (Story 21-5 AC-9) — pages eusolicit-test-burn-rate-alert
    — verification: test page received as low-urgency in PagerDuty (no mobile push)
[ ] Real-page calibration: manual PagerDuty "New Incident" test to eusolicit-platform-prod
    — verification: on-call engineer receives mobile push within 30s
```

---

## §Runbook Inventory + URL Coverage Audit

### Runbook Inventory

All 15 runbooks in `eusolicit-docs/runbooks/` (12 new in PE.06 + 3 pre-existing from other stories):

| Runbook | SLA-Scope | Severity | PE.05 `runbook_url` |
|---------|-----------|----------|---------------------|
| `error-budget-burn.md` | in-scope | SEV-1/2 | `alerting-rules.yaml` lines 39–48 + 60–68 |
| `high-latency.md` | in-scope | SEV-2 | `alerting-rules.yaml` lines 73–82 |
| `rds-replica-lag.md` | in-scope | SEV-1/2 | `alerting-rules.yaml` lines 85–96 |
| `redis-evictions.md` | in-scope | SEV-2 | `alerting-rules.yaml` lines 99–124 (×2 alerts) |
| `kraftdata-outage.md` | EXEMPT (vendor outage) | SEV-2 | `kraftdata-isolation-rules.yaml` line 49 |
| `pg-failover.md` | in-scope | SEV-1/2 | — (operator runbook, not alert-linked) |
| `redis-failover.md` | in-scope | SEV-1/2 | — (operator runbook, not alert-linked) |
| `node-drain.md` | in-scope | SEV-2/3 | — (operator runbook, not alert-linked) |
| `stripe-outage.md` | EXEMPT (vendor outage) | SEV-2/3 | — |
| `clamav-outage.md` | EXEMPT (vendor outage) | SEV-2 | — |
| `ingress-controller-restart.md` | in-scope | SEV-2/3 | — |
| `full-disk-on-pg.md` | in-scope | SEV-1/2 | — |
| `oauth-provider-outage.md` | EXEMPT (vendor outage) | SEV-2 | — |
| `bulk-webhook-replay.md` | in-scope | SEV-2 | — |
| `deploy-rollback.md` | in-scope | SEV-1/2 | — |
| `sub-processor-change.md` | — | — | Pre-existing; Story 18-2; UNTOUCHED |

### URL Coverage Audit

```
CI lint gate result (run: python3 scripts/check_runbook_url_coverage.py):
✅ PE.06 runbook coverage lint PASSED — runbook coverage: 7/7 URLs resolve, 5/5 structural checks pass

ATDD test result (run: pytest tests/unit/test_runbook_url_coverage.py -v):
✅ 15/15 passed

URL-to-file mapping (7 annotations across 2 rule files → 5 unique runbook files):
  alerting-rules.yaml:
    line 48:  error-budget-burn.md  ✅ exists + all sections populated
    line 68:  error-budget-burn.md  ✅ (duplicate — SustainedErrorBudgetBurnRate also links here)
    line 82:  high-latency.md       ✅ exists + all sections populated
    line 96:  rds-replica-lag.md    ✅ exists + all sections populated
    line 110: redis-evictions.md    ✅ exists + all sections populated
    line 124: redis-evictions.md    ✅ (duplicate — RedisReplicationLagHigh also links here)
  kraftdata-isolation-rules.yaml:
    line 49:  kraftdata-outage.md   ✅ exists + all sections populated
```

### SLA-Scope Carve-Out Verification

```
Explicit EXEMPT carve-outs (anti-pattern guard #5: NEVER omit SLA-Scope from any runbook):
  kraftdata-outage.md: **SLA-Scope**: EXEMPT (vendor outage) ✅
  stripe-outage.md:    **SLA-Scope**: EXEMPT (vendor outage) ✅
  clamav-outage.md:    **SLA-Scope**: EXEMPT (vendor outage) ✅
  oauth-provider-outage.md: **SLA-Scope**: EXEMPT (vendor outage) ✅

In-scope runbooks:
  error-budget-burn.md:         **SLA-Scope**: in-scope ✅
  high-latency.md:              **SLA-Scope**: in-scope ✅
  rds-replica-lag.md:           **SLA-Scope**: in-scope ✅
  redis-evictions.md:           **SLA-Scope**: in-scope ✅
  pg-failover.md:               **SLA-Scope**: in-scope ✅
  redis-failover.md:            **SLA-Scope**: in-scope ✅
  node-drain.md:                **SLA-Scope**: in-scope ✅
  ingress-controller-restart.md:**SLA-Scope**: in-scope ✅
  full-disk-on-pg.md:           **SLA-Scope**: in-scope ✅
  bulk-webhook-replay.md:       **SLA-Scope**: in-scope ✅
  deploy-rollback.md:           **SLA-Scope**: in-scope ✅
```

---

## §Incident Process Verification

### Process Documents Inventory

| File | Purpose | Story AC |
|------|---------|----------|
| `eusolicit-docs/incident-management/severity-definitions.md` | SEV-1/2/3 definitions + response SLAs + SLA-scope table + Alertmanager label mapping | AC-3 |
| `eusolicit-docs/incident-management/incident-response-process.md` | 9-step IC-led incident flow; Slack `#platform-incidents` format; PagerDuty ack procedure | AC-3 |
| `eusolicit-docs/incident-management/post-mortem-template.md` | Blameless post-mortem template; mirrors Epic 13 retro structure; no "who" column in Timeline | AC-4 |
| `eusolicit-docs/incident-management/status-page-comms-templates.md` | SEV-1 Initial/Update/Resolved + EXEMPT vendor incident templates; no-blame phrasing table | AC-4 |

### Severity ↔ Alertmanager Label Mapping

| Alertmanager `severity` label | PagerDuty urgency | SEV classification | Response target |
|-------------------------------|-------------------|--------------------|-----------------|
| `page` | high | SEV-1 or SEV-2 | 5 min ack (SEV-1) / 15 min ack (SEV-2) |
| `ticket` | low | SEV-2 or SEV-3 | 15 min ack (SEV-2) / next business day (SEV-3) |
| `info` | not routed to PD | SEV-3 | Next business day |

### Blameless Culture Invariants

The post-mortem template enforces blamelessness structurally:
- **No "who" column** in the Timeline table — only "what" and "why"
- **Action Items table** has an "Owner" column for remediation assignment, not for blame
- **Exemplar**: `eusolicit-docs/implementation-artifacts/epic-13-retro-2026-04-26.md` — the project-context retrospective that established the blameless pattern for this project
- **Rule**: Root cause analysis targets systems and processes, never individuals

---

## §Chaos-Drill Post-Mortem Reference

### First Post-Mortem

The PE.04 chaos drill (Story 21-4 `pe-04-chaos-drill-runbook.md` §Per-Service Drill) is treated as the first simulated incident per Epic 21 line 153. The post-mortem skeleton is committed at:

```
eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md
```

This is the **first entry** in the new `eusolicit-docs/post-mortems/` directory. Future real-incident post-mortems use `eusolicit-docs/incident-management/post-mortem-template.md` and are committed to the same directory.

### Operator Action Required

```
D-3 pre-recorded deviation: The operator MUST:
1. Execute the PE.04 chaos drill per pe-04-chaos-drill-runbook.md §Per-Service Drill
   against the staging environment using the runbooks authored in this story.
2. Rename the skeleton: 2026-MM-DD-pe-04-chaos-drill.md → 2026-<actual-date>-pe-04-chaos-drill.md
3. Populate the post-mortem skeleton with actual drill data:
   - YAML frontmatter: incident_id, start_time, detect_time, ack_time, resolve_time, drill_rota per service
   - Executive Summary paragraphs
   - Timeline rows
   - What Went Well / What Went Poorly
   - Action Items (runbook gaps become sprint stories)
   - Lessons Learned paragraphs
4. Sign off with IC + Drill Reviewer + CTO
5. Commit the populated post-mortem to eusolicit-docs/post-mortems/
```

### Runbook Coverage During Drill

The drill MUST exercise these runbooks (per AC-7):
- `node-drain.md` (primary drill runbook — all 6 services drained sequentially)
- `redis-failover.md` (if Redis pod is on a drained node)
- `pg-failover.md` (N/A for node drain — AWS Multi-AZ failover is separate; optional during drill)

Runbooks NOT exercised during the drill should be flagged as "not yet exercised by drill" in the post-mortem §What Went Poorly and scheduled for the next drill rotation.

---

## §2-Week Soak Gate

Per Epic 21 line 155: the public 99.9% SLA announcement is gated on the on-call schedule being active for ≥2 weeks with at least one real page received.

```
Soak start date: [OPERATOR RECORDS — date of first Terraform apply to staging/prod]
First real page: [OPERATOR RECORDS — date of first PagerDuty incident or Story 21-5 AC-9 synthetic page]
Soak end date (≥2 weeks after start): [OPERATOR RECORDS]
Public SLA announcement unblocked: [YES / NO]

Calibration page: Story 21-5 AC-9 synthetic burn-rate test fires a page to
eusolicit-test-burn-rate-alert (low-urgency). The first real page to eusolicit-platform-prod
(high-urgency) is the operational proof point. Both count toward the 2-week soak evidence.
```

---

## §Sign-off

| Role | Name | Date |
|------|------|------|
| Dev Agent (autopilot) | bmad-dev-story | 2026-05-05 |
| Operator (after soak) | `<PLACEHOLDER>` | `<YYYY-MM-DD>` |
| CTO | `<PLACEHOLDER>` | `<YYYY-MM-DD>` |

> **Gate note**: The dev-pass closes with Status: review. The `done` gate requires:
> (1) bmad-code-review **Approve** verdict (AP17-C1 two-gate-close),
> (2) Operator completes D-3 soak gate (2-week on-call rotation active + first real page received),
> (3) Operator populates and signs the chaos-drill post-mortem,
> (4) Epic Review (ER) confirms all PE.01–PE.06 stories closed.
> Public 99.9% SLA announcement is unblocked only after all four gates pass.
