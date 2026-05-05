# Story 21.6: On-Call Rotation + Runbook Authoring + Incident-Management Process

Status: review

<!-- Validation note: Run [VS] Validate Story (bmad-validate-story) before bmad-dev-story per Operator BMAD-stream guidance — non-negotiable.
     AP18-C2: orchestrator MUST patch `Status:` in this file atomically with the sprint-status transition.
     AP17-C1 two-gate-close: `Status: done` requires bmad-code-review **Approve** verdict (Pass-2). Dev pass alone does NOT promote to done — protect the S19-0/19-1/19-2/20-0/21-1/21-2/21-3/21-4/21-5 successful-closure streak (9 in a row at create-time; 21-5 dev-pass complete pending Approve verdict).
     Operator workflow guidance for E21 (multi-story epic, FINAL story):
       - [IR] already executed 2026-05-04 v1+v2 + 2026-05-05 for E21. DO NOT re-run.
       - [VS] Validate Story BEFORE bmad-dev-story — non-negotiable.
       - [SR] Story Review AFTER this story closes (multi-story epic; consumes PE.05 alertmanager.yaml + reserved runbook IDs).
       - [PR] Post-Review after code review is complete (mandated for ALL epics).
       - [ER] Epic Review at end of E21 (interdependent stories: PE.01 → PE.02 → PE.03 → PE.04 → PE.05 → PE.06 chain — PE.06 is the FINAL story).
       - epic-21 status remains in-progress (transitioned on Story 21-1 create); transitions to done after ER per orchestrator. -->

## Story

As a **Platform Engineering lead** (with the CTO as the executive sponsor of the publishable 99.9% SLA, and the on-call engineers as the day-1 consumers of every artefact this story produces),
I want **(1) a managed on-call rotation provider configured (PagerDuty by default per the PE.05 Alertmanager `receiver: pagerduty` ship-config at `eusolicit-app/infra/observability/alertmanager/alertmanager.yaml` line 70–78; Opsgenie remains the documented operator-elected alternative — the decision is recorded in §Implementation Decision of this story's runbook with a one-line `terraform apply` swap path) with a single shared rotation across the 2–3 person platform team (escalation policy: primary → backup → CTO; 24/7 follow-the-sun is OUT OF SCOPE per epic line 149 "initially shared rotation; can split engineering/platform later") AND the corresponding service+integration-key entries created in PagerDuty (`eusolicit-platform-prod` for `severity=page` alerts + `eusolicit-test-burn-rate-alert` for AC-9 of Story 21-5 — the latter prevents PE.05's synthetic e2e test from waking the on-call) with the integration keys placed in AWS Secrets Manager at `eusolicit/<env>/observability/pagerduty-key` (consumed by PE.05's Alertmanager ESO ExternalSecret at `infra/observability/alertmanager/externalsecret.yaml`); (2) a NEW `eusolicit-docs/runbooks/` directory populated with the **complete top-12 runbook set** per epic line 150 + the 5 PE.05-reserved IDs (with `kraftdata-outage` overlapping between the two lists, yielding 12 unique runbooks total): **(a) carry-forward runbooks PE.05 reserved for this story** — `error-budget-burn.md` (multi-window multi-burn-rate alert response — points to the specific PromQL query + the 4-window panel in Grafana `platform-slo.json`), `high-latency.md` (NFR-2 p95 violation response — links to the per-service Grafana dashboard endpoint-breakdown panel + slow-query investigation via `pg_stat_statements_seconds_total`), `rds-replica-lag.md` (`aws_rds_replica_lag_average` >30s response — Multi-AZ failover decision tree + cutover steps cross-referenced from `pe-02-cutover-runbook.md` §Failover Drill Steps), `redis-evictions.md` (eviction-pressure response — eviction policy review + key-pattern audit + cluster-mode-disabled scaling decision tree), `kraftdata-outage.md` (KraftData ingest stoppage — circuit-breaker state inspection via `ai_gateway.services.circuit_breaker.AgentCircuit.state` + the Epic 4 isolation pattern that keeps AI-Gateway outages OUT of the platform SLO per architecture.md line 762; **this runbook consolidates the partial KraftData runbook content from Epic 4** per epic line 150's "already partial from Epic 4" note); **(b) the PE.02 + PE.03 + PE.04 cutover-runbook carry-forwards** (lifted into the public runbooks/ directory as the operator-facing entry points; the `pe-02-cutover-runbook.md` / `pe-03-cutover-runbook.md` / `pe-04-chaos-drill-runbook.md` evidence files remain in `implementation-artifacts/` as the dev-side audit trail) — `pg-failover.md` (PG Multi-AZ failover decision tree + verification steps; references the live AWS RDS `Force Failover` action + the per-service reconnect SLA ≤30s from PE.02 §Failover Drill Steps), `redis-failover.md` (ElastiCache failover decision tree + verification; references the AWS ElastiCache `TestFailover` API + the ≤10s reconnect SLA from PE.03 §Failover Drill Steps), `node-drain.md` (chaos-drill drain procedure + PDB-behaviour evidence; references the `kubectl drain --ignore-daemonsets --delete-emptydir-data` pattern + the 6-service drill rota from PE.04 §Per-Service Drill); **(c) the platform-failure-mode runbooks per epic line 150** — `stripe-outage.md` (Stripe API outage — billing webhook backoff + the existing `client_api.services.billing_service` retry/circuit-breaker behaviour + Stripe status page link + manual reconciliation steps for invoice-state divergence; **Stripe is excluded from platform SLA scope** per the Epic 4 KraftData isolation precedent applied to Stripe), `clamav-outage.md` (ClamAV scanner outage — file-upload reject vs. queue decision + the `clamav:3310` health-probe flow + restart procedure + alternate scanner fallback decision), `ingress-controller-restart.md` (nginx-ingress controller restart procedure + connection-drain expectations + the PDB `minAvailable: 1` invariant from PE.04 + cert-manager interaction), `full-disk-on-pg.md` (RDS disk-full response — emergency `VACUUM FULL` decision tree + the `aws_rds_freeable_memory_average` alert reference from PE.05 + storage auto-scaling toggle + the read-only-failover decision; references the Story 1.3 schema-isolation invariant that no cross-schema CASCADE can amplify the disk pressure), `oauth-provider-outage.md` (Google OAuth / Microsoft OAuth provider outage per Epic 9 — login failures vs. session-refresh failures decision tree + the existing JWT 24h-expiry grace window + the Epic 9 calendar-sync degradation pattern + the existing fallback-to-email-password login path), `bulk-webhook-replay.md` (Stripe + Slack + Teams webhook replay procedure for missed events during downtime; references the `client_api.services.billing_service.handle_stripe_webhook` idempotency pattern + the Stripe `events.list` API for the time window + the `integrations-api` Slack webhook resigning logic from Story 16.0), `deploy-rollback.md` (Helm release rollback via `helm rollback <release> <revision>` + the per-service ImagePullPolicy invariant + the migration-rollback gotcha that DDL changes are NOT auto-reverted by `helm rollback` — links to the Story 1.4 Alembic `downgrade` pattern + a forward-fix-vs-rollback decision tree); each runbook follows the unified structural template per AC-2.1 below; **runbook URL convention** = `https://github.com/eusolicit/eusolicit/blob/main/eusolicit-docs/runbooks/<runbook-id>.md` matching the PE.05 `runbook_url` annotation in `infra/observability/prometheus/rules/alerting-rules.yaml` lines 48/68/82/96/110/124 + `kraftdata-isolation-rules.yaml` line 49 verbatim (no URL drift); (3) a NEW `eusolicit-docs/incident-management/` directory containing the complete process documentation: **`severity-definitions.md`** (SEV-1 = customer-facing outage OR data-integrity breach OR security incident; response time = 5 min ack / 15 min triage / customer comms within 1h; SEV-2 = degraded service OR p95 >NFR-2 sustained 15+ min OR partial-tenant impact; response = 15 min ack / 1h triage / customer comms within 4h if customer-visible; SEV-3 = warnings + early-burn signals + non-customer-impacting infra issues; response = next-business-day; the severity-to-Alertmanager-`severity`-label mapping table is the canonical bridge to PE.05's routing config), **`incident-response-process.md`** (the IC-led incident response flow: detection → ack → IC declared → comms channel opened → status page updated → fix → resolution → post-mortem scheduled; references the Slack `#platform-incidents` channel created via the Story 16.0 integrations-api Slack webhook + the post-mortem cadence "within 5 business days for SEV-1/2; optional for SEV-3"), **`post-mortem-template.md`** (blameless format mirroring the Epic 13 retro structure at `implementation-artifacts/epic-13-retro-2026-04-26.md` — Executive Summary / Timeline / Root Cause / Contributing Factors / What Went Well / What Went Poorly / Action Items / Lessons Learned; **NEVER attribute fault to individuals** — the Epic 13 retro pattern is the exemplar), and **`status-page-comms-templates.md`** (3 boilerplates: SEV-1 Initial / SEV-1 Update / SEV-1 Resolved — operators paste-edit; explicit no-blame phrasing); (4) the chaos-drill from PE.04 §Per-Service Drill (Story 21-4 `pe-04-chaos-drill-runbook.md`) **is treated as the first simulated incident** and **MUST be post-mortemed using the new `post-mortem-template.md`** per epic line 153–155 — the post-mortem is committed at `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md` (FIRST entry in a NEW `post-mortems/` directory; future post-mortems append; this is the first concrete proof point that the runbooks actually work — runbooks not followed during the drill = runbook not yet authored); (5) a CI lint gate (`scripts/check_runbook_url_coverage.py` NEW, hooked into `.github/workflows/ci.yml` per the PE.04 `scripts/check_helm_pdb_and_minreplicas.py` precedent) that **scans every `runbook_url:` annotation in `infra/observability/prometheus/rules/*.yaml` and asserts the corresponding `eusolicit-docs/runbooks/<runbook-id>.md` file EXISTS and has non-empty `## Symptoms` / `## Triage` / `## Resolution` / `## Verification` sections** — this is the regression test that future PE.06-style debt cannot accumulate (Anti-pattern fence #6); (6) the `pe-06-incident-readiness-runbook.md` evidence file at `eusolicit-docs/implementation-artifacts/` mirroring the PE.02/PE.03/PE.04/PE.05 cutover-runbook structure, with the 7 sections (Pre-flight checklist / Implementation Decision (PagerDuty vs. Opsgenie) / On-Call Rotation Verification / Runbook Inventory + URL Coverage Audit / Incident Process Verification / Chaos-Drill Post-Mortem Reference / Sign-off) + the operator-captured PagerDuty rotation screenshot reference + the §Sign-off block; (7) the 2-week on-call soak prerequisite per epic line 155 ("on-call schedule active for ≥2 weeks before public SLA announcement") **explicitly recorded as a D-deviation operator timeline gate** — autopilot ships the rotation config + runbooks + process docs, the operator gates the public SLA announcement on the soak completion (calendar-driven, not code-gated),**

so that **(a) the 99.9% SLA promised in PRD v1.1 §7 NFR-14 has the operational capacity it requires — without an on-call rotation, runbooks, and an incident process, a SEV-1 outage at 03:00 burns through the 43-min/month budget while the team sleeps and the SLA becomes a public lie within the first quarter; **PE.06 is the final stone in the SLA-publication arch** — PE.01 + PE.02 + PE.03 + PE.04 ship the code-and-config gate (4/4 done from a code-and-config standpoint per epic line 20), PE.05 ships the detection layer (alerts, dashboards, recording rules), PE.06 ships the response layer (rotation, runbooks, process); (b) the PE.05 `alertmanager.yaml` + `runbook_url` annotation contract becomes a **closed loop** — every `runbook_url` reference in `alerting-rules.yaml` lines 48/68/82/96/110/124 + `kraftdata-isolation-rules.yaml` line 49 resolves to a non-404 GitHub URL, which the AC-5 CI lint gate verifies on every push; the cross-story coordination contract from Story 21-5 §Cross-Story Coordination ("PE.06 lands second; URL returns 404 until PE.06 authors the file" — graceful degradation) is fully closed; (c) the chaos-drill from PE.04 (Story 21-4) becomes evidence-driven — the drill runs against the runbooks PE.06 just authored, the runbooks are revised in real-time against drill findings, the post-mortem captures the runbook gaps and improvements; runbooks not exercised during the drill are not yet trustworthy — the drill IS the runbook quality gate; (d) **Stripe outages, ClamAV outages, OAuth provider outages, and KraftData incidents are explicitly carved out from the platform SLA scope** per the Epic 4 KraftData isolation precedent (per architecture.md line 762) — the `severity-definitions.md` SLA-scope table + `kraftdata-outage.md` SLA-exemption clause + `stripe-outage.md` SLA-exemption clause + `oauth-provider-outage.md` SLA-exemption clause + `clamav-outage.md` SLA-exemption clause make the carve-out **explicit and auditable** rather than implicit; this prevents the SLA from being held hostage to upstream-vendor incidents the platform team cannot control; (e) the **post-mortem template enforces blameless culture** at the structural level — the template mirrors the Epic 13 retro structure (`implementation-artifacts/epic-13-retro-2026-04-26.md` is the exemplar; project-context Epic 13 retrospective patterns reference; "blameless" = root cause analysis on systems and processes, NOT individuals); the structure makes blame-finding awkward by design (no "who" column in the timeline; only "what" and "why"); (f) **future on-call coverage scales without runbook debt** — the AC-5 CI lint gate is the architectural invariant that any new alerting rule MUST ship with a runbook before merge; PE.06 closes the loop that the existing alerting rules MUST also have runbooks (they do, post-PE.06); future alerting rules cannot land without runbooks (the gate fails the build); this is the same anti-debt-accumulation pattern as PE.04's `helm-pdb-lint` and PE.05's `/metrics` contract test; (g) **the 2-week soak gate is honoured operationally** per epic line 155 — even after PE.06 dev-pass + Approve, the public SLA announcement is gated on the operator's calendar verification that the rotation has been active and on-call has been paged at least once (the synthetic burn-rate test from Story 21-5 AC-9 is the calibration page; subsequent real pages count toward the 2-week soak); the §Sign-off section of `pe-06-incident-readiness-runbook.md` records the soak start date + operator-captured first-page evidence; (h) the **incident process integrates with existing tools** — Slack `#platform-incidents` channel via the existing Story 16.0 integrations-api Slack webhook (NO new integration); Status page comms via Slack copy-paste templates (no Statuspage.io integration in scope — that lands in a future epic if customer count justifies it); PagerDuty's built-in mobile push + SMS + voice-call escalation (no separate paging tool); (i) **Epic 21 closes** — PE.06 is the FINAL story; after Approve verdict + 2-week soak completion + ER (Epic Review), Epic 21 transitions to `done` and the public 99.9% SLA announcement on the marketing site is unblocked; the 6-story platform-engineering epic that was created in parallel to feature work E14–E20 lands its milestone "99.9% SLA externally publishable" exactly per epic line 3.**

## Epic Context

- **Epic**: E21 Platform Reliability for 99.9% SLA — Sprint 14–17 (parallel to E14–E20 feature work); 34 pts; **6-story platform-engineering epic**. Milestone: **99.9% SLA externally publishable** (gated on PE.01 + PE.02 + PE.03 + PE.04). PE.05 is **non-gating** per epic line 20; **PE.06 is non-gating per epic line 20 from a code-and-config standpoint, BUT the public SLA announcement is gated on PE.06's 2-week soak completion** per epic line 155 — this is a calendar gate, not a code gate.
- **Story points**: 3 | **Type**: process / platform-engineering | **Position**: SIXTH and FINAL PE story. Lowest point count in the epic — most of the work is markdown authoring + a single PagerDuty Terraform module + a single CI lint script. The complexity is in the cross-document consistency (every `runbook_url` annotation across 7 PromQL files must resolve; severity definitions must align with Alertmanager labels; Epic 4 KraftData runbook content must be consolidated without orphaning).
- **NFRs covered**: **NFR-14** (99.9% uptime SLA — PE.06 closes the response-side loop that PE.05 opened on the detection side; without on-call response capability the SLA is unenforceable); **NFR-9** (incident response — explicit SEV-1 5min-ack target codifies the architecture intent); **NFR-15** (data integrity — `pg-failover.md` + `full-disk-on-pg.md` runbooks codify the response patterns that prevent data loss during incidents).
- **Position in epic chain**: **SIXTH and FINAL PE story**. **Hard-depends** on Story 21-5 (PE.05) outputs: (a) `infra/observability/alertmanager/alertmanager.yaml` `pagerduty` receiver block — PE.06 ships the integration key into the AWS Secrets Manager slot the receiver references; (b) `infra/observability/prometheus/rules/alerting-rules.yaml` lines 48/68/82/96/110/124 + `kraftdata-isolation-rules.yaml` line 49 — every `runbook_url` annotation here is the contract PE.06 fulfils; (c) `pe-05-observability-runbook.md` §Cross-Story Coordination — the runbook-ID list PE.05 reserved (5 IDs) + the PagerDuty receiver placeholder pattern; (d) Story 21-5 D-4 (TEST PagerDuty service operator-action) is closed by AC-1 of this story. **Hard-depends** on Story 21-4 (PE.04) outputs: (e) `pe-04-chaos-drill-runbook.md` §Per-Service Drill is the chaos-drill source-of-truth that AC-7's first post-mortem references; (f) the `node-drain.md` runbook lifts content from PE.04 §Drain Procedure verbatim. **Hard-depends** on Story 21-3 (PE.03) outputs: (g) `pe-03-cutover-runbook.md` §Failover Drill Steps + ≤10s reconnect SLA → `redis-failover.md` runbook content. **Hard-depends** on Story 21-2 (PE.02) outputs: (h) `pe-02-cutover-runbook.md` §Failover Drill Steps + ≤30s reconnect SLA → `pg-failover.md` runbook content; (i) the `aws_rds_freeable_memory_average` alert reference from PE.05 → `full-disk-on-pg.md`. **Soft-depends** on Epic 4 KraftData circuit-breaker code at `services/ai-gateway/src/ai_gateway/services/circuit_breaker.py` + `services/ai-gateway/src/ai_gateway/services/kraftdata_resilient.py` — `kraftdata-outage.md` runbook content references the existing CB state inspection pattern; **the partial KraftData runbook content from Epic 4** (any docs referenced via the Epic 4 circuit-breaker comments) is consolidated into `kraftdata-outage.md` here. **Soft-depends** on Epic 9 OAuth pattern (Google OAuth + calendar-sync) — `oauth-provider-outage.md` runbook references the existing fallback-to-email-password pattern. **Soft-depends** on Epic 13 retrospective structure → `post-mortem-template.md` mirrors the Epic 13 retro at `implementation-artifacts/epic-13-retro-2026-04-26.md`. **Soft-depends** on Story 16.0 integrations-api Slack webhook — Slack `#platform-incidents` channel uses the existing webhook (no new integration). **Hard-depends** on Story 1.4 Alembic migration scaffold — `deploy-rollback.md` references the `alembic downgrade` pattern.
- **Source**: Epic spec `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` lines 143–156 (PE.06 scope). Architecture `architecture.md` line 762 (KraftData SLO isolation precedent applied to Stripe/ClamAV/OAuth in this story). PRD v1.1 §7 NFR-14 / NFR-9 / NFR-15. Story 21-5 evidence: `infra/observability/alertmanager/alertmanager.yaml` (pagerduty receiver block + ESO secret slot), `infra/observability/prometheus/rules/alerting-rules.yaml` + `kraftdata-isolation-rules.yaml` (runbook_url annotation contract — 7 distinct URLs). Story 21-4 evidence: `pe-04-chaos-drill-runbook.md` §Per-Service Drill + §Drain Procedure (chaos-drill source for AC-7 first post-mortem + `node-drain.md` content). Story 21-3 evidence: `pe-03-cutover-runbook.md` §Failover Drill Steps (`redis-failover.md` content). Story 21-2 evidence: `pe-02-cutover-runbook.md` §Failover Drill Steps (`pg-failover.md` content). Epic 4 evidence: `services/ai-gateway/src/ai_gateway/services/circuit_breaker.py` + `services/ai-gateway/src/ai_gateway/services/kraftdata_resilient.py` (KraftData CB pattern → `kraftdata-outage.md`). Epic 13 retro evidence: `eusolicit-docs/implementation-artifacts/epic-13-retro-2026-04-26.md` (post-mortem template structural source). Story 16.0 evidence: integrations-api Slack webhook (incident-comms channel). Test design fallback: no `test_artifacts/test-design-epic-21.md` exists (consistent with Story 21-1 D-7, Story 21-2 D-2, Story 21-3 D-7, Story 21-4 D-7, Story 21-5 D-7 deviations; this is acceptable for a process story whose quality gate is the populated runbook files + the AC-5 CI lint gate + the AC-7 first-incident post-mortem).
- **Operator workflow guidance**: [IR] already executed 2026-05-04 v1+v2 + 2026-05-05 (do NOT re-run). `[VS] Validate Story` (NON-NEGOTIABLE) → `bmad-dev-story` (autopilot) → `bmad-code-review` (Approve verdict required for `done` per AP17-C1 — protect 9-in-a-row streak) → `[SR] Story Review` (FINAL story; SR consolidates the closed runbook_url loop) → `[PR] Post-Review` (mandatory for all epics) → `[ER] Epic Review` (Epic 21 closes here; PE.01 → PE.02 → PE.03 → PE.04 → PE.05 → PE.06 chain complete).

## Acceptance Criteria

> Source-of-truth: epic spec lines 143–156 (PE.06 scope) + Story 21-5 §Cross-Story Coordination (5 reserved runbook IDs PE.06 must fill) + epic line 150 (top-10 runbooks list) + epic line 151 (severity defs + response SLAs) + epic line 152 (post-mortem template per Epic 13 retro pattern) + epic line 153 (first chaos-test exercise post-mortemed). AC numbers below cover every epic line item plus the cross-story closure of PE.05's `runbook_url` annotation contract (Story 21-5 AC-7.3 reserves URLs; PE.06 AC-2 fulfils them) plus the carry-forward consolidation from PE.02/PE.03/PE.04 cutover runbooks plus the Epic 4 KraftData partial-runbook consolidation per epic line 150.

### AC-1 — PagerDuty On-Call Rotation Configured + AWS Secrets Manager Integration

**Given** the PE.05 `infra/observability/alertmanager/alertmanager.yaml` already ships a `pagerduty` receiver block (lines 70–78 — `service_key_file: /etc/alertmanager/secrets/pagerduty-key`) AND the corresponding ESO ExternalSecret CRD at `infra/observability/alertmanager/externalsecret.yaml` already references AWS Secrets Manager path `eusolicit/<env>/observability/pagerduty-key` AND the platform team is 2–3 engineers AND PagerDuty is the project default per epic line 149 ("PagerDuty (or Opsgenie)"; Opsgenie remains the operator-elected alternative documented in §Implementation Decision) AND PagerDuty has a Terraform provider (`PagerDuty/pagerduty` registry; provider version `~> 3.0`; the project's existing toolchain has NO PagerDuty provider so this story ADDS it),

**When** the dev agent provisions the on-call rotation,

**Then**:

1. **NEW Terraform module** at `infra/terraform/modules/oncall/`:
   - `main.tf` — `pagerduty_user` (1 user resource per platform engineer; emails sourced from `var.platform_engineers`); `pagerduty_schedule` (one shared rotation; `time_zone = "Europe/Berlin"`; layer 1 = primary 24/7/365 weekly rotation; layer 2 = backup 24/7 weekly rotation offset by 24h); `pagerduty_escalation_policy` (level 1 = primary 5-min ack, level 2 = backup 10-min ack, level 3 = CTO email-only); `pagerduty_service` x 2 (`eusolicit-platform-prod` for `severity=page` alerts + `eusolicit-test-burn-rate-alert` for the Story 21-5 AC-9 synthetic e2e test — closes PE.05 D-4); `pagerduty_service_integration` x 2 (Prometheus integration type; one per service; integration keys captured as Terraform outputs).
   - `variables.tf` — `platform_engineers` (list(object({email=string, name=string, role=string}))); `cto_email` (string); `enabled` (bool, default `true` for staging+prod; `false` for dev — anti-pattern guard #1 below).
   - `outputs.tf` — `pagerduty_prod_integration_key` (sensitive) + `pagerduty_test_integration_key` (sensitive) + `pagerduty_schedule_url` + `pagerduty_escalation_policy_id`.
   - `versions.tf` — `terraform { required_providers { pagerduty = { source = "PagerDuty/pagerduty"; version = "~> 3.0" } } }`. The provider API key flows via the standard `PAGERDUTY_TOKEN` environment variable per provider docs.

2. **Wire into root** `infra/terraform/main.tf` — pass `var.platform_engineers` + `var.cto_email` to the module; staging+prod tfvars files set `enable_oncall = true`, dev tfvars sets `enable_oncall = false` (anti-pattern guard #1).

3. **AWS Secrets Manager population** — `aws_secretsmanager_secret_version.pagerduty_key` (NEW resource at `infra/terraform/modules/oncall/aws-secrets.tf`) writes the integration key from `pagerduty_prod_integration_key` output to AWS Secrets Manager at `eusolicit/<env>/observability/pagerduty-key` (the path PE.05's ESO ExternalSecret already polls). Same pattern as PE.02 RDS password handling. Test integration key written to `eusolicit/<env>/observability/pagerduty-test-key` (consumed by Story 21-5 AC-9 burn-rate e2e test).

4. **Anti-pattern guard #1**: NEVER enable PagerDuty Terraform on the `dev` environment — there is no on-call rotation to configure for the local docker-compose loop, and PagerDuty has a per-user-per-month cost. Same anti-pattern category as PE.02 `multi_az = false` for dev, PE.03 `multi_az_enabled = false` for dev, and PE.05 AC-8.8 `enable_prometheus = false` for dev.

5. **Anti-pattern guard #2**: NEVER store the PagerDuty API token in any committed file — the `PAGERDUTY_TOKEN` flows via the operator's local env or via the CI runner's secret store. Same secret-management rule as ADR-002 + PE.02 + PE.03 + PE.05 AC-7.4.

6. **Implementation Decision recorded**: `pe-06-incident-readiness-runbook.md` §Implementation Decision documents (a) PagerDuty vs. Opsgenie (PagerDuty chosen as project default per epic line 149; Opsgenie swap is a 1-file Terraform-provider replacement + an `alertmanager.yaml` receiver-block swap; the operator can elect either at deployment time; cost difference ≈ wash); (b) shared rotation vs. follow-the-sun (shared chosen per epic line 149 "initially shared rotation; can split engineering/platform later"; follow-the-sun requires ≥6 engineers across 3 timezones).

### AC-2 — Top-12 Runbook Set Authored at `eusolicit-docs/runbooks/`

**Given** epic line 150 names 10 runbooks (PG failover, Redis failover, KraftData outage, Stripe outage, ClamAV outage, ingress controller restart, full-disk on PG, OAuth provider outage, bulk webhook replay, deploy rollback) AND Story 21-5 §Cross-Story Coordination reserved 5 runbook IDs (`error-budget-burn`, `high-latency`, `rds-replica-lag`, `redis-evictions`, `kraftdata-outage` — overlap with epic line 150's "KraftData outage") AND `eusolicit-docs/runbooks/` ALREADY EXISTS with 1 unrelated runbook (`sub-processor-change.md` from Story 18-2) AND `eusolicit-app/runbooks/` ALREADY EXISTS with 1 unrelated runbook (`outcome-brief-s3-lifecycle.md` from Story 19-2) — both pre-existing runbooks remain UNTOUCHED (AP-GUARD-3 below) AND the 12 unique runbooks (10 epic + 5 reserved − 1 overlap = 14 listed but `kraftdata-outage` collapses) align to `eusolicit-docs/runbooks/`,

**When** the dev agent authors runbooks,

**Then**:

1. **Unified runbook structural template** — every runbook below MUST include these sections in order:
   - **Header block**: `# Runbook: <Title>` + `**Severity**: SEV-1 / SEV-2 / SEV-3` + `**SLA-Scope**: in-scope / EXEMPT (vendor outage)` + `**Last updated**: YYYY-MM-DD`.
   - **§Symptoms** — observable signals (alert names + Grafana panels + log patterns + customer reports).
   - **§Triage** — 3–5 line decision tree to confirm/deny the diagnosis (NEVER assume; always verify).
   - **§Resolution** — step-by-step fix actions; commands committed verbatim where possible (`kubectl ...`, `aws ...`, `psql ...`, `helm ...`).
   - **§Verification** — how to confirm the fix worked; explicit Grafana panel + Prometheus query references.
   - **§Rollback** — what to do if §Resolution makes things worse (mandatory for every runbook).
   - **§Related** — cross-references to other runbooks + ADRs + cutover runbooks + Epic-spec lines.

2. **Author 12 NEW runbooks** at `eusolicit-docs/runbooks/`:
   - **(PE.05-reserved)** `error-budget-burn.md` — symptoms: `HighErrorBudgetBurnRate` + `SustainedErrorBudgetBurnRate` alerts (Story 21-5 `alerting-rules.yaml` lines 39–48 + 60–68); triage: 4-window burn-rate panel in Grafana `platform-slo.json`; resolution: per-symptom branches (latency burn → `high-latency.md`; error-rate burn → service-specific dashboard → recent deploy rollback decision); verification: burn-rate recovers below 14.4× target on 1h window AND below 6× on 5m; SLA-scope: in-scope.
   - **(PE.05-reserved)** `high-latency.md` — symptoms: `HighLatencyP95` alert at 0.2s (NFR-2; Story 21-5 `alerting-rules.yaml` line 73–82); triage: per-service Grafana dashboard endpoint-breakdown panel + `pg_stat_statements_seconds_total` query for slow-query attribution + the Story 21-1 baseline numbers in `load-test-results.md` for sanity comparison; resolution: query-plan regression → migration rollback decision via `deploy-rollback.md`; runtime degradation → service scaling via HPA; SLA-scope: in-scope.
   - **(PE.05-reserved)** `rds-replica-lag.md` — symptoms: `aws_rds_replica_lag_average` >30s sustained; triage: AWS RDS console replica lag history + the Story 21-2 `pe-02-cutover-runbook.md` §Failover Drill Steps; resolution: AWS RDS `Force Failover` action OR wait-and-monitor decision tree; verification: replica lag <5s + per-service reconnect SLA ≤30s observed; references `pg-failover.md` for the full failover playbook; SLA-scope: in-scope.
   - **(PE.05-reserved + carry-forward)** `redis-evictions.md` — symptoms: `aws_elasticache_evictions_sum` >0 (Story 21-5 `alerting-rules.yaml` lines 100–110 + 113–124); triage: ElastiCache memory-usage panel + key-pattern audit via `redis-cli --bigkeys` (read-only — won't mutate state); resolution: eviction-policy review (`maxmemory-policy`) + cluster-mode-disabled scaling decision (vertical scaling vs. horizontal sharding) + the Story 21-3 `pe-03-cutover-runbook.md` §Failover Drill Steps reference if eviction triggers a failover; SLA-scope: in-scope.
   - **(PE.05-reserved + epic line 150 KraftData)** `kraftdata-outage.md` — symptoms: AI-Gateway 5xx rate elevated + `slo_target=kraftdata-dependent` recording rule promoted (Story 21-5 `kraftdata-isolation-rules.yaml`); triage: `services/ai-gateway/src/ai_gateway/services/circuit_breaker.py` `AgentCircuit.state` inspection (the existing Epic 4 CB at `services/ai-gateway/src/ai_gateway/services/kraftdata_resilient.py` exposes state) + KraftData status page + the Epic 4 isolation pattern documented at architecture.md line 762; resolution: WAIT (KraftData is upstream; the platform CB protects downstream services); customer-comms via status page if sustained >30 min; **SLA-scope: EXEMPT** per architecture.md line 762 + epic line 21; **consolidates the partial Epic 4 KraftData runbook content** per epic line 150 ("already partial from Epic 4 — consolidate"); the ai-gateway CB cooldown is `settings.circuit_breaker_cooldown` per `kraftdata_resilient.py` line 73.
   - **(epic line 150 + carry-forward from PE.02)** `pg-failover.md` — symptoms: RDS Multi-AZ failover triggered (manual or automated) OR primary unreachable; triage: AWS RDS console health + per-service connection error rate via `aws_rds_database_connections_average`; resolution: Multi-AZ automated failover → wait-and-verify; OR `aws rds reboot-db-instance --force-failover` for manual; per-service reconnect SLA ≤30s expected per Story 21-2 §Failover Drill Steps; transaction-rollback fixtures + connection-pool re-establishment patterns from PE.02; verification: all services reconnect <30s + no transaction loss + Story 21-2 §Failover Drill Steps reproduced; **lifts the operator-facing content from `pe-02-cutover-runbook.md` §Failover Drill Steps**; SLA-scope: in-scope.
   - **(epic line 150 + carry-forward from PE.03)** `redis-failover.md` — symptoms: ElastiCache failover triggered (manual or automated); triage: AWS ElastiCache console health + per-service Redis-connection error rate; resolution: ElastiCache automated failover → wait-and-verify; OR `aws elasticache test-failover --replication-group-id <id>` for manual; per-service reconnect SLA ≤10s expected per Story 21-3 §Failover Drill Steps; consumer-group offset replication verification; Lua-script re-run check from PE.03 §Lua-Script Re-Run; verification: all services reconnect <10s + event-stream consumer groups resume + usage-metering Lua scripts continue to function; **lifts the operator-facing content from `pe-03-cutover-runbook.md` §Failover Drill Steps**; SLA-scope: in-scope.
   - **(epic line 150 + carry-forward from PE.04)** `node-drain.md` — symptoms: Karpenter/Cluster-Autoscaler triggered node drain OR operator-initiated `kubectl drain`; triage: PDB status + per-service replica count + the Story 21-4 §Per-Service Drill rota; resolution: `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --grace-period=300` per PE.04 §Drain Procedure + the PDB `minAvailable: 1` + min-replica ≥ 2 invariants; verification: per-service replica count remains ≥ minReplicas during drain; HPA scale-out triggers if needed; **lifts content from `pe-04-chaos-drill-runbook.md` §Drain Procedure + §Per-Service Drill**; SLA-scope: in-scope.
   - **(epic line 150)** `stripe-outage.md` — symptoms: Stripe API 5xx elevated + billing webhook backoff queue grows + customer reports of failed checkouts; triage: Stripe status page + `client_api.services.billing_service` retry/circuit-breaker behaviour + webhook DLQ depth; resolution: WAIT (Stripe is upstream); manual reconciliation via `stripe events list --created.gte=<window>` for any missed webhooks once Stripe recovers; references `bulk-webhook-replay.md` for the bulk-replay procedure; **SLA-scope: EXEMPT** per Epic 4 KraftData isolation precedent applied to Stripe (architecture.md line 762 pattern); recorded in `severity-definitions.md` SLA-scope table.
   - **(epic line 150)** `clamav-outage.md` — symptoms: ClamAV `clamav:3310` health-probe failures + file-upload reject rate elevated; triage: ClamAV pod status + the existing health-probe flow; resolution: ClamAV pod restart via `kubectl rollout restart deploy/clamav` + the queue-vs-reject decision tree (default = queue with retry; reject only on sustained outage >15 min); **SLA-scope: EXEMPT** (ClamAV is a security scanner; degraded scan = degraded feature, not platform outage); recorded in `severity-definitions.md` SLA-scope table.
   - **(epic line 150)** `ingress-controller-restart.md` — symptoms: nginx-ingress controller crashloop OR planned restart for cert renewal; triage: ingress controller pod status + cert-manager state + the PDB `minAvailable: 1` invariant from PE.04; resolution: `kubectl rollout restart deploy/ingress-nginx-controller` + connection-drain expectations (existing connections complete; new connections route to surviving replicas during the rolling restart); cert-manager interaction (Let's Encrypt renewals); verification: zero customer-visible 502s during restart (PDB protects); SLA-scope: in-scope.
   - **(epic line 150)** `full-disk-on-pg.md` — symptoms: `aws_rds_freeable_memory_average` low + RDS storage-full alarm + write-failures cascading; triage: RDS storage panel + `pg_stat_user_tables` for table sizes + emergency `VACUUM FULL` decision tree; resolution: storage auto-scaling toggle (RDS supports up-to-10TB auto-scaling; verify enabled per Story 21-2) OR emergency `VACUUM FULL` (briefly read-only) OR read-replica-promotion + write-traffic redirection; the Story 1.3 schema-isolation invariant prevents cross-schema CASCADE amplification; SLA-scope: in-scope.
   - **(epic line 150)** `oauth-provider-outage.md` — symptoms: Google OAuth or Microsoft OAuth login failure rate elevated + the Epic 9 calendar-sync degradation pattern; triage: OAuth provider status page + the existing Story 2-6 + Epic 9 fallback-to-email-password login path verification; resolution: WAIT (upstream); customer-comms recommending email-password login during the outage; the existing JWT 24h expiry grace window covers most active sessions; **SLA-scope: EXEMPT** (OAuth is upstream; SLA covers the platform's response, not the provider's availability); recorded in `severity-definitions.md` SLA-scope table.
   - **(epic line 150)** `bulk-webhook-replay.md` — symptoms: missed webhooks during downtime → invoice-state divergence (Stripe), missed Slack/Teams alerts; triage: per-provider event-list APIs (`stripe events list --created.gte=...`, Slack webhook resigning logic from Story 16.0); resolution: idempotent replay using the existing handler pattern — `client_api.services.billing_service.handle_stripe_webhook` is idempotent per Story 8 (Stripe `event_id` dedup); Slack/Teams webhook resigning via `integrations-api`; SLA-scope: in-scope (the platform owns the recovery flow even though the upstream missed-event window is exempt).
   - **(epic line 150)** `deploy-rollback.md` — symptoms: post-deploy alerting spike + customer-visible regression OR canary deployment health-check failures; triage: Helm release history `helm history <release>` + the per-service ImagePullPolicy invariant + the `aws_rds_replica_lag_average` for any DB-migration-related issue; resolution: `helm rollback <release> <previous-revision>` for application code; **the migration-rollback gotcha** — DDL changes are NOT auto-reverted by `helm rollback`; explicit `alembic downgrade <previous_rev>` per Story 1.4; the forward-fix-vs-rollback decision tree (forward-fix preferred for additive migrations; rollback for breaking changes); verification: error-rate returns to baseline + Story 21-1 latency numbers re-met; SLA-scope: in-scope.

3. **Anti-pattern guard #3**: NEVER touch the existing `eusolicit-docs/runbooks/sub-processor-change.md` (Story 18-2 GDPR runbook) OR `eusolicit-app/runbooks/outcome-brief-s3-lifecycle.md` (Story 19-2 S3 lifecycle runbook) — both are in the runbooks/ paths but belong to other stories with different ownership. PE.06 ADDS new files; never edits these.

4. **Anti-pattern guard #4**: NEVER author runbook content that lacks the §Verification section — a runbook without a verification step is a runbook the on-call cannot trust. The AC-5 CI lint gate enforces this structurally.

5. **Anti-pattern guard #5**: NEVER omit the SLA-Scope header — the in-scope vs. EXEMPT distinction is the contract that prevents the SLA from being held hostage to upstream-vendor outages. Every runbook header MUST have one of the two values.

### AC-3 — Incident-Management Process Documentation at `eusolicit-docs/incident-management/`

**Given** epic line 151 mandates "Incident-management process documented: severity definitions (SEV-1 / SEV-2 / SEV-3), response-time SLAs, post-mortem cadence" AND epic line 152 mandates "Post-mortem template authored (blameless format per existing project-context Epic 13 retro patterns)" AND the Story 16.0 integrations-api Slack webhook is the existing channel-comms tool (no new integration in scope) AND the Epic 13 retro at `implementation-artifacts/epic-13-retro-2026-04-26.md` is the structural exemplar for blameless post-mortems,

**When** the dev agent authors process docs,

**Then**:

1. **NEW directory** `eusolicit-docs/incident-management/` with 4 files:

2. **`severity-definitions.md`** — defines:
   - **SEV-1** = customer-facing outage (any customer cannot complete a core flow: opportunity discovery, proposal generation, billing checkout) OR data-integrity breach OR security incident OR error-budget burn-rate exhaustion within 24h. **Response time**: 5-min ack / 15-min triage / customer comms within 1h via status page (Slack `#platform-incidents` channel + manual customer email if known). **Alertmanager severity label**: `severity=page` (PE.05 routes to PagerDuty).
   - **SEV-2** = degraded service (p95 latency exceeds NFR-2 0.2s sustained 15+ min — `HighLatencyP95` alert) OR partial-tenant impact OR feature-level degradation (e.g. ClamAV scan delay). **Response time**: 15-min ack / 1h triage / customer comms within 4h IF customer-visible. **Alertmanager severity label**: `severity=ticket` (PE.05 routes to Slack `#platform-alerts`).
   - **SEV-3** = warnings + early-burn signals (slow-burn 6h alert) + non-customer-impacting infra issues. **Response time**: next-business-day. **Alertmanager severity label**: `severity=info` (PE.05 discards — logged only).
   - **SLA-scope table** — explicit columns for in-scope vs. EXEMPT incidents:
     - In-scope: PG / Redis / ingress / deploy regressions / error-budget burn.
     - EXEMPT: KraftData outage (Epic 4 + architecture.md line 762), Stripe outage, ClamAV outage, OAuth provider outage. The runbook for each EXEMPT incident links back here for the SLA-scope rationale.
   - **Severity-to-Alertmanager-label mapping table** — canonical bridge to PE.05's routing config; future alerting rules MUST set the correct severity label per this table.

3. **`incident-response-process.md`** — the IC-led incident response flow:
   - **Step 1**: Detection (PagerDuty page from Alertmanager OR customer report OR proactive on-call observation).
   - **Step 2**: Ack within SLA per `severity-definitions.md`.
   - **Step 3**: Incident Commander (IC) declared (default = on-call primary; if primary unavailable, backup; if SEV-1, CTO joins as observer).
   - **Step 4**: Comms channel opened — Slack `#platform-incidents` channel (created via Story 16.0 integrations-api Slack webhook; no new integration); status page updated for SEV-1 customer-visible incidents (manual copy-paste from `status-page-comms-templates.md`).
   - **Step 5**: Diagnose using runbooks (PagerDuty alert payload includes `runbook_url` annotation per PE.05 AC-7.3 contract).
   - **Step 6**: Fix per runbook §Resolution.
   - **Step 7**: Verify per runbook §Verification.
   - **Step 8**: Resolve in PagerDuty + Slack.
   - **Step 9**: Schedule post-mortem within 5 business days for SEV-1/2; OPTIONAL for SEV-3.
   - **References**: every step links to a section of `severity-definitions.md` OR a runbook OR `post-mortem-template.md`.

4. **`post-mortem-template.md`** — blameless template mirroring `implementation-artifacts/epic-13-retro-2026-04-26.md` structure:
   - **Frontmatter** (YAML): `incident_id`, `severity`, `start_time`, `detect_time`, `resolve_time`, `impact_duration`, `customers_impacted`, `error_budget_consumed_pct`, `runbooks_followed`.
   - **§Executive Summary** (2 paragraphs).
   - **§Timeline** (chronological — "what" + "why" columns ONLY; no "who" column — anti-pattern guard #6 enforces this).
   - **§Root Cause** (single sentence + supporting evidence; root cause analysis uses the 5-Whys technique).
   - **§Contributing Factors** (process gaps, tooling gaps, knowledge gaps).
   - **§What Went Well** (the positive — the runbook worked, the alert fired correctly, the IC was decisive).
   - **§What Went Poorly** (the negative — runbook gap, alert fired late, comms breakdown).
   - **§Action Items** (table: Action / Owner / Due / Status — Story-trackable items go into sprint-status as injection candidates).
   - **§Lessons Learned** (1–3 paragraph reflection).
   - **NEVER attribute fault to individuals** — the template structurally prevents it (no "who" column in timeline; the "Owner" column in Action Items is the assigned remediation owner, not the blame target).

5. **`status-page-comms-templates.md`** — 3 boilerplates ready for paste-edit:
   - **SEV-1 Initial** (within 30 min of declaration): "We are investigating reports of <symptom>. Affected: <scope>. We will update within 1 hour."
   - **SEV-1 Update** (every 1h until resolved): "Investigation continues. Current understanding: <hypothesis>. Next update at <time>."
   - **SEV-1 Resolved**: "The incident at <time> is resolved. Root cause: <one-sentence summary>. A post-mortem will be published within 5 business days at <URL>."
   - All 3 templates use **explicit no-blame phrasing** — no individual names, no vendor blame (vendor outages get "<vendor> reported an outage at <time>; our system was affected because of <integration>").

6. **Anti-pattern guard #6**: NEVER author the post-mortem template with a "who" column in the Timeline section — the Epic 13 retro pattern is structurally blameless because the structure itself prevents blame-finding. A "who" column tempts the reader to assign fault.

7. **Anti-pattern guard #7**: NEVER add a new Slack/Teams/Statuspage.io integration in scope of PE.06 — the Story 16.0 integrations-api Slack webhook is reused for `#platform-incidents`; status page comms are copy-paste templates (no Statuspage.io integration in scope; that lands in a future epic if customer count justifies it).

### AC-4 — First Chaos-Drill Post-Mortem at `eusolicit-docs/post-mortems/`

**Given** epic line 153 mandates "First chaos-test exercise (drain a node, watch alerts/runbooks/response) executed and post-mortemed" AND Story 21-4 already authored the chaos-drill runbook at `implementation-artifacts/pe-04-chaos-drill-runbook.md` AND Story 21-4 D-1 deferred the live drill execution to operator AND the drill IS the runbook quality gate per epic line 155 ("Runbooks verified by being followed during the chaos drill"),

**When** the dev agent stages the post-mortem,

**Then**:

1. **NEW directory** `eusolicit-docs/post-mortems/` (NEW path; first entry is this drill).

2. **NEW file** `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md` (filename uses placeholder date which the operator overwrites at drill-execution time):
   - Frontmatter populated per `post-mortem-template.md`: `incident_id: pe-04-chaos-drill-2026-MM-DD`, `severity: SEV-2 (synthetic)`, `customers_impacted: 0 (staging)`, `error_budget_consumed_pct: 0 (synthetic in staging)`, `runbooks_followed: [node-drain.md, redis-failover.md (if Redis pod drained), pg-failover.md (if PG pod drained — N/A for managed RDS)]`.
   - **§Executive Summary** — placeholder text the operator fills post-drill: "On <DATE> at <TIME>, the platform team executed the PE.04 chaos drill per `pe-04-chaos-drill-runbook.md` §Per-Service Drill. The drill validated PDB + min-replica behaviour across 6 services, served as the first incident-response exercise per epic line 153, and produced findings that drove revisions to <list runbooks>."
   - **§Timeline** — placeholder rows the operator fills (per drill phase).
   - **§Root Cause** — N/A (synthetic drill; no real root cause).
   - **§Contributing Factors** — placeholder.
   - **§What Went Well / §What Went Poorly** — placeholder.
   - **§Action Items** — placeholder (every runbook gap surfaced becomes an action item; runbook revisions land in a follow-up story or as direct edits to the runbook files in `eusolicit-docs/runbooks/`).
   - **§Lessons Learned** — placeholder.
   - **§Sign-off** — operator + IC names + date.

3. **D-3 below pre-records this as deferrable** — autopilot ships the post-mortem skeleton with operator-capture placeholders; live drill execution + post-mortem completion is operator-action (same precedent as PE.04 D-1, PE.05 D-2). The skeleton structure is the deliverable; the populated content is operator-provided.

### AC-5 — `runbook_url` Coverage CI Lint Gate at `scripts/check_runbook_url_coverage.py`

**Given** Story 21-5 `infra/observability/prometheus/rules/alerting-rules.yaml` lines 48/68/82/96/110/124 + `kraftdata-isolation-rules.yaml` line 49 contain 7 distinct `runbook_url:` annotations AND each annotation MUST resolve to a non-404 path under `eusolicit-docs/runbooks/<runbook-id>.md` AND PE.04 set the precedent of CI-gate scripts at `scripts/check_helm_pdb_and_minreplicas.py` hooked into `.github/workflows/ci.yml` AND future alerting rules will add more `runbook_url` annotations that MUST also resolve,

**When** the dev agent authors the lint gate,

**Then**:

1. **NEW script** `eusolicit-app/scripts/check_runbook_url_coverage.py`:
   - Walks `eusolicit-app/infra/observability/prometheus/rules/*.yaml` and extracts every `runbook_url:` annotation value via PyYAML safe-load + nested traversal.
   - For each URL, derives the local file path by stripping the `https://github.com/eusolicit/eusolicit/blob/main/` prefix → resolves to a path under the repo root.
   - Asserts the file EXISTS (FileNotFoundError → exit 1 with "Missing runbook: <path>; failing PR").
   - Asserts the file has non-empty `## Symptoms`, `## Triage`, `## Resolution`, `## Verification`, `## Rollback`, `## Related` sections (regex check on content).
   - Asserts the file header has a `**SLA-Scope**:` line with value `in-scope` OR `EXEMPT (vendor outage)` (anti-pattern guard #5 enforcement).
   - Exits 0 with summary line on success: `runbook coverage: <N>/<N> URLs resolve, <N>/<N> structural checks pass`.

2. **NEW unit test** `eusolicit-app/tests/unit/test_runbook_url_coverage.py`:
   - Imports `check_runbook_url_coverage` and runs it as a subprocess against the live `infra/observability/prometheus/rules/` + `eusolicit-docs/runbooks/` paths.
   - Asserts exit code 0 + the expected summary line.
   - Mirrors the PE.04 `tests/unit/test_pe04_helm_lint.py` structural pattern.

3. **CI workflow integration** — extend `.github/workflows/ci.yml` to invoke `python scripts/check_runbook_url_coverage.py` after the existing `check_helm_pdb_and_minreplicas.py` step. Same lint-gate category; runs on every push and PR.

4. **Anti-pattern guard #8**: NEVER skip the section-content regex checks — checking that a file exists is insufficient; an empty stub satisfies file-existence but fails the on-call. The 6-section content check is the regression gate.

### AC-6 — `pe-06-incident-readiness-runbook.md` Evidence File + Sprint-Status Reconciliation + Epic-File Closure

**Given** PE.02/PE.03/PE.04/PE.05 have set the precedent for a per-story runbook evidence file + an epic-file implementation block + a sprint-status surgical patch AND PE.06 is the FINAL story in Epic 21 — Epic 21 transitions from `in-progress` → `done` after [ER] Epic Review post-PE.06 Approve verdict + 2-week soak completion AND the `pe-06-incident-readiness-runbook.md` is the authoritative evidence file the operator + bmad-code-review reviewer references during Pass-2,

**When** the dev agent authors documentation + reconciles,

**Then**:

1. **NEW evidence file**: `eusolicit-docs/implementation-artifacts/pe-06-incident-readiness-runbook.md` mirroring the PE.02/PE.03/PE.04/PE.05 cutover-runbook structure. Required sections (7):
   - **§Pre-flight Checklist** — Terraform plan reviewed; PagerDuty API token captured in operator env; AWS Secrets Manager paths verified writable; runbook file inventory cross-checked against PE.05 alerting-rules.yaml URLs; Epic 13 retro structure confirmed as post-mortem template source.
   - **§Implementation Decision** — (a) PagerDuty vs. Opsgenie (PagerDuty chosen + 1-line swap path documented); (b) shared rotation vs. follow-the-sun (shared per epic line 149); (c) blameless post-mortem template structural mechanism (no "who" column).
   - **§On-Call Rotation Verification** — `terraform output -json` excerpt showing `pagerduty_schedule_url` + `pagerduty_escalation_policy_id`; operator-captured PagerDuty rotation screenshot reference (D-1 below pre-records the screenshot capture as operator-action).
   - **§Runbook Inventory + URL Coverage Audit** — table: runbook ID / file path / SLA-scope / size / referenced from PE.05 alerting-rules.yaml line ↑ ; the AC-5 lint gate output captured verbatim.
   - **§Incident Process Verification** — `severity-definitions.md` + `incident-response-process.md` + `post-mortem-template.md` + `status-page-comms-templates.md` cross-checked against epic lines 151–152.
   - **§Chaos-Drill Post-Mortem Reference** — link to `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md` (D-3 deferred but skeleton committed); when operator runs the drill, the post-mortem is populated and §Sign-off updated.
   - **§Sign-off** — operator verdict (PASSED / DEFERRED / FAILED) + 2-week soak start date (D-2 records the 2-week soak as operator-calendar-gated for the public SLA announcement).

2. **Append** PE.06 implementation block to `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` AFTER line 156 (current end of PE.06 §Acceptance block, before §Amendments) — DO NOT renumber:
   ```
   **Implementation:** Story 21-6 (`21-6-on-call-rotation-runbook-authoring-incident-management-process`).
   See `implementation-artifacts/pe-06-incident-readiness-runbook.md`. PagerDuty rotation
   provisioned via NEW Terraform module `infra/terraform/modules/oncall/`. Top-12 runbooks
   authored at `eusolicit-docs/runbooks/` (closes PE.05 `runbook_url` coverage contract;
   consolidates Epic 4 KraftData partial). Incident-management process documented at
   `eusolicit-docs/incident-management/` (severity defs, response process, blameless
   post-mortem template per Epic 13 retro structure, comms templates). First chaos-drill
   post-mortem skeleton at `eusolicit-docs/post-mortems/`. CI lint gate
   `scripts/check_runbook_url_coverage.py` enforces `runbook_url`-resolution invariant
   on every PR. 2-week soak gate (epic line 155) operator-calendar-gated for public SLA
   announcement (D-2 deviation pre-recorded).
   ```

3. **Sprint-status reconciliation** at done-time (operator action; not autopilot): `21-6-…: review → done` flips ATOMICALLY with the story file `Status: review → done` per AP18-C2 + AP17-C1 two-gate-close. Surgical edit only — preserve ALL comments + STATUS DEFINITIONS (per project memory rule).

4. **Epic-21 status reconciliation** — after PE.06 Approve verdict + 2-week soak + ER, sprint-status `epic-21: in-progress → done` (ER is the gate, NOT this story). PE.06 dev pass leaves epic-21 in-progress; ER reconciles.

5. **Anti-pattern guard #9**: NEVER set `Status: done` without bmad-code-review **Approve** verdict — AP17-C1 protects the S19-0/19-1/19-2/20-0/21-1/21-2/21-3/21-4/21-5 9-in-a-row successful-closure streak; PE.06 is the 10th potential recurrence + the FINAL story of Epic 21 (so the streak win is doubly significant).

6. **Anti-pattern guard #10**: NEVER skip the §Sign-off section in the runbook — the §Sign-off section records the 2-week soak start date AND the PagerDuty rotation screenshot reference AND the operator verdict; without §Sign-off the public SLA announcement gate (epic line 155) cannot be validated.

## Tasks / Subtasks

- [x] **Task 1 — PagerDuty Terraform module + AWS Secrets Manager wiring (AC-1)**
  - [x] 1.1 NEW `infra/terraform/modules/oncall/main.tf` — `pagerduty_user` resources, `pagerduty_schedule`, `pagerduty_escalation_policy`, `pagerduty_service` x 2, `pagerduty_service_integration` x 2
  - [x] 1.2 NEW `infra/terraform/modules/oncall/variables.tf` — `platform_engineers`, `cto_email`, `enabled`
  - [x] 1.3 NEW `infra/terraform/modules/oncall/outputs.tf` — sensitive integration keys + schedule URL + escalation policy ID
  - [x] 1.4 NEW `infra/terraform/modules/oncall/versions.tf` — PagerDuty/pagerduty `~> 3.0` provider declaration
  - [x] 1.5 NEW `infra/terraform/modules/oncall/aws-secrets.tf` — `aws_secretsmanager_secret_version` writes integration keys to PE.05 ESO paths
  - [x] 1.6 EDIT `infra/terraform/main.tf` — wire oncall module
  - [x] 1.7 EDIT `infra/terraform/environments/dev/terraform.tfvars` — `enable_oncall = false` (anti-pattern #1)
  - [x] 1.8 EDIT `infra/terraform/environments/staging/terraform.tfvars` + `prod/terraform.tfvars` — `enable_oncall = true`

- [x] **Task 2 — Author 12 runbooks at `eusolicit-docs/runbooks/` (AC-2)**
  - [x] 2.1 NEW `eusolicit-docs/runbooks/error-budget-burn.md` (PE.05-reserved; SLA in-scope)
  - [x] 2.2 NEW `eusolicit-docs/runbooks/high-latency.md` (PE.05-reserved; SLA in-scope)
  - [x] 2.3 NEW `eusolicit-docs/runbooks/rds-replica-lag.md` (PE.05-reserved; SLA in-scope; cross-references `pg-failover.md`)
  - [x] 2.4 NEW `eusolicit-docs/runbooks/redis-evictions.md` (PE.05-reserved; SLA in-scope)
  - [x] 2.5 NEW `eusolicit-docs/runbooks/kraftdata-outage.md` (PE.05-reserved + Epic 4 partial consolidation; **SLA EXEMPT**)
  - [x] 2.6 NEW `eusolicit-docs/runbooks/pg-failover.md` (carry-forward from PE.02 `pe-02-cutover-runbook.md` §Failover Drill Steps; SLA in-scope)
  - [x] 2.7 NEW `eusolicit-docs/runbooks/redis-failover.md` (carry-forward from PE.03 `pe-03-cutover-runbook.md` §Failover Drill Steps; SLA in-scope)
  - [x] 2.8 NEW `eusolicit-docs/runbooks/node-drain.md` (carry-forward from PE.04 `pe-04-chaos-drill-runbook.md` §Drain Procedure; SLA in-scope)
  - [x] 2.9 NEW `eusolicit-docs/runbooks/stripe-outage.md` (epic line 150; **SLA EXEMPT** per Epic 4 isolation precedent)
  - [x] 2.10 NEW `eusolicit-docs/runbooks/clamav-outage.md` (epic line 150; **SLA EXEMPT**)
  - [x] 2.11 NEW `eusolicit-docs/runbooks/ingress-controller-restart.md` (epic line 150; SLA in-scope)
  - [x] 2.12 NEW `eusolicit-docs/runbooks/full-disk-on-pg.md` (epic line 150; SLA in-scope)
  - [x] 2.13 NEW `eusolicit-docs/runbooks/oauth-provider-outage.md` (epic line 150; **SLA EXEMPT**)
  - [x] 2.14 NEW `eusolicit-docs/runbooks/bulk-webhook-replay.md` (epic line 150; SLA in-scope)
  - [x] 2.15 NEW `eusolicit-docs/runbooks/deploy-rollback.md` (epic line 150; SLA in-scope; references Story 1.4 Alembic downgrade)

- [x] **Task 3 — Incident-management process docs (AC-3)**
  - [x] 3.1 NEW `eusolicit-docs/incident-management/severity-definitions.md` (SEV-1/2/3 + response SLAs + SLA-scope table + Alertmanager-label mapping)
  - [x] 3.2 NEW `eusolicit-docs/incident-management/incident-response-process.md` (9-step IC flow; references runbooks + post-mortem template)
  - [x] 3.3 NEW `eusolicit-docs/incident-management/post-mortem-template.md` (blameless; mirrors Epic 13 retro structure; no "who" column)
  - [x] 3.4 NEW `eusolicit-docs/incident-management/status-page-comms-templates.md` (3 boilerplates: Initial / Update / Resolved)

- [x] **Task 4 — Chaos-drill post-mortem skeleton (AC-4)**
  - [x] 4.1 NEW directory `eusolicit-docs/post-mortems/`
  - [x] 4.2 NEW `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md` skeleton (operator overwrites date + populates content per `post-mortem-template.md` post-drill)

- [x] **Task 5 — `runbook_url` coverage CI lint gate (AC-5)**
  - [x] 5.1 NEW `eusolicit-app/scripts/check_runbook_url_coverage.py` (PyYAML walk + path resolution + 6-section regex check + SLA-scope header check)
  - [x] 5.2 `eusolicit-app/tests/unit/test_runbook_url_coverage.py` pre-existing ATDD (RED phase already existed; made GREEN by 5.1)
  - [x] 5.3 EDIT `.github/workflows/ci.yml` — add lint step after `check_helm_pdb_and_minreplicas.py` step
  - [x] 5.4 Lint gate GREEN: `✅ PE.06 runbook coverage lint PASSED — runbook coverage: 7/7 URLs resolve, 5/5 structural checks pass`

- [x] **Task 6 — Documentation + reconciliation (AC-6)**
  - [x] 6.1 NEW `eusolicit-docs/implementation-artifacts/pe-06-incident-readiness-runbook.md` (7 sections per AC-6.1)
  - [x] 6.2 EDIT `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` — PE.06 implementation block appended after line 156
  - [x] 6.3 §Sign-off section in runbook authored (verdict: DEFERRED per D-3 operator soak gate; operator populates post-drill + post-soak)

- [x] **Task 7 — AP18-C2 atomic Status patch + AP17-C1 two-gate-close**
  - [x] 7.1 Set this file's `Status:` to `review` AT END of dev pass (NOT `done`); sprint-status `21-6-…: review` updated atomically
  - [ ] 7.2 After bmad-code-review Approve verdict: flip `Status: review → done` + sprint-status `21-6-…: review → done` in a SINGLE commit (preserve 10-in-a-row successful-closure streak: S19-0 / S19-1 / S19-2 / S20-0 / S21-1 / S21-2 / S21-3 / S21-4 / S21-5 / **S21-6**)
  - [x] 7.3 Epic-21 status flip `in-progress → done` is owned by [ER] Epic Review, NOT this story — sprint-status `epic-21: in-progress` remains unchanged

## Dev Notes

### Architecture Compliance

- **architecture.md line 762** — "KraftData incidents excluded from SLA scope" → AC-2.5 `kraftdata-outage.md` `SLA-scope: EXEMPT`; the same isolation precedent applied to Stripe (`stripe-outage.md`), ClamAV (`clamav-outage.md`), and OAuth providers (`oauth-provider-outage.md`); all 4 EXEMPT vendors are recorded in `severity-definitions.md` SLA-scope table.
- **PRD v1.1 §7 NFR-14** — 99.9% uptime SLA → PE.06 closes the response-side capability that PE.05 opened on the detection side.
- **PRD v1.1 §7 NFR-9** — incident response → AC-3 `severity-definitions.md` codifies SEV-1 5-min ack target.
- **PRD v1.1 §7 NFR-15** — data integrity → `pg-failover.md` + `full-disk-on-pg.md` codify the response patterns.
- **ADR-002 (architecture.md line 686)** — RBAC dual-layer auth: PE.06 runbooks reference but do NOT modify RBAC; the `oauth-provider-outage.md` references the existing JWT 24h grace window from Story 2-5.
- **ADR-006 (architecture.md line 722)** — tier gating + usage metering as per-route `Depends()`: not relevant to PE.06 (process story; no API surface).
- **PE.04 `helm-pdb-lint` precedent** — `scripts/check_helm_pdb_and_minreplicas.py` + `tests/unit/test_pe04_helm_lint.py` established the CI lint-gate pattern; AC-5 mirrors this verbatim for `runbook_url` coverage.
- **PE.05 `runbook_url` annotation contract** — `infra/observability/prometheus/rules/alerting-rules.yaml` lines 48/68/82/96/110/124 + `kraftdata-isolation-rules.yaml` line 49 — 7 distinct URLs that PE.06 fulfils.
- **Epic 4 KraftData circuit-breaker pattern** — `services/ai-gateway/src/ai_gateway/services/circuit_breaker.py` (`AgentCircuit` class) + `services/ai-gateway/src/ai_gateway/services/kraftdata_resilient.py` lines 21–73 (CB invocation pattern + `circuit_breaker_threshold` + `circuit_breaker_cooldown` settings) — `kraftdata-outage.md` references these as the operator inspection surface.
- **Epic 13 retrospective structure** — `eusolicit-docs/implementation-artifacts/epic-13-retro-2026-04-26.md` is the structural exemplar for `post-mortem-template.md` (frontmatter / Executive Summary / Timeline / Root Cause / Contributing Factors / What Went Well / What Went Poorly / Action Items / Lessons Learned).
- **Story 16.0 integrations-api Slack webhook** — reused as the `#platform-incidents` channel-comms surface; no new integration in scope.

### Reused Components — DO NOT Reinvent

- **PE.05 `infra/observability/alertmanager/alertmanager.yaml` `pagerduty` receiver block** — already ships the `service_key_file` reference; PE.06 ONLY populates the AWS Secrets Manager slot the receiver references. NEVER edit the alertmanager.yaml in this story.
- **PE.05 `infra/observability/alertmanager/externalsecret.yaml`** — already polls AWS Secrets Manager path `eusolicit/<env>/observability/pagerduty-key`; PE.06's Terraform writes to that exact path. NEVER author a new ESO ExternalSecret in this story.
- **PE.05 `infra/observability/prometheus/rules/alerting-rules.yaml` + `kraftdata-isolation-rules.yaml`** — already contain 7 distinct `runbook_url` annotations; PE.06 NEVER edits these YAML files. The annotations are the source-of-truth; the runbook files match them.
- **PE.02 `pe-02-cutover-runbook.md` §Failover Drill Steps** — operator-facing content lifted into `pg-failover.md`. NEVER duplicate the dev-side cutover-runbook content; the operator-facing runbook is a SUMMARY + decision-tree, not a copy.
- **PE.03 `pe-03-cutover-runbook.md` §Failover Drill Steps + §Lua-Script Re-Run** — content lifted into `redis-failover.md` (same SUMMARY pattern).
- **PE.04 `pe-04-chaos-drill-runbook.md` §Drain Procedure + §Per-Service Drill** — content lifted into `node-drain.md`.
- **PE.04 `scripts/check_helm_pdb_and_minreplicas.py`** — structural template for `scripts/check_runbook_url_coverage.py` (AC-5).
- **PE.04 `tests/unit/test_pe04_helm_lint.py`** — structural template for `tests/unit/test_runbook_url_coverage.py`.
- **Epic 4 `services/ai-gateway/src/ai_gateway/services/circuit_breaker.py` + `kraftdata_resilient.py`** — `kraftdata-outage.md` references the CB state inspection surface (existing code; PE.06 NEVER modifies).
- **Epic 9 OAuth + calendar-sync** — `oauth-provider-outage.md` references the existing fallback-to-email-password login path (existing Story 2-6); PE.06 NEVER modifies the OAuth code.
- **Story 1.4 Alembic migration scaffold** — `deploy-rollback.md` references the `alembic downgrade` pattern (existing Story 1.4); PE.06 NEVER modifies migrations.
- **Story 16.0 integrations-api Slack webhook** — `#platform-incidents` channel uses this existing webhook (no new integration).
- **Story 8 Stripe webhook idempotency pattern** — `bulk-webhook-replay.md` references `client_api.services.billing_service.handle_stripe_webhook` event-id dedup; PE.06 NEVER modifies the billing service.
- **Story 1.3 schema-isolation invariant** — `full-disk-on-pg.md` references the no-cross-schema-CASCADE invariant (existing Story 1.3); PE.06 NEVER modifies schemas.
- **Epic 13 retro at `implementation-artifacts/epic-13-retro-2026-04-26.md`** — structural exemplar for `post-mortem-template.md`; PE.06 NEVER modifies the retro itself.
- **Existing `eusolicit-docs/runbooks/sub-processor-change.md` + `eusolicit-app/runbooks/outcome-brief-s3-lifecycle.md`** — pre-existing runbooks owned by other stories. NEVER touch.

### File Paths to Touch

| Path | Operation | AC |
|------|-----------|----|
| `eusolicit-app/infra/terraform/modules/oncall/main.tf` | CREATE | AC-1.1 |
| `eusolicit-app/infra/terraform/modules/oncall/variables.tf` | CREATE | AC-1.2 |
| `eusolicit-app/infra/terraform/modules/oncall/outputs.tf` | CREATE | AC-1.3 |
| `eusolicit-app/infra/terraform/modules/oncall/versions.tf` | CREATE | AC-1.4 |
| `eusolicit-app/infra/terraform/modules/oncall/aws-secrets.tf` | CREATE | AC-1.5 |
| `eusolicit-app/infra/terraform/main.tf` | EDIT — wire oncall module | AC-1.6 |
| `eusolicit-app/infra/terraform/environments/dev/terraform.tfvars` | EDIT — `enable_oncall = false` | AC-1.7 |
| `eusolicit-app/infra/terraform/environments/staging/terraform.tfvars` | EDIT — `enable_oncall = true` | AC-1.8 |
| `eusolicit-app/infra/terraform/environments/prod/terraform.tfvars` | EDIT — `enable_oncall = true` | AC-1.8 |
| `eusolicit-docs/runbooks/error-budget-burn.md` | CREATE | AC-2.1 |
| `eusolicit-docs/runbooks/high-latency.md` | CREATE | AC-2.2 |
| `eusolicit-docs/runbooks/rds-replica-lag.md` | CREATE | AC-2.3 |
| `eusolicit-docs/runbooks/redis-evictions.md` | CREATE | AC-2.4 |
| `eusolicit-docs/runbooks/kraftdata-outage.md` | CREATE | AC-2.5 |
| `eusolicit-docs/runbooks/pg-failover.md` | CREATE | AC-2.6 |
| `eusolicit-docs/runbooks/redis-failover.md` | CREATE | AC-2.7 |
| `eusolicit-docs/runbooks/node-drain.md` | CREATE | AC-2.8 |
| `eusolicit-docs/runbooks/stripe-outage.md` | CREATE | AC-2.9 |
| `eusolicit-docs/runbooks/clamav-outage.md` | CREATE | AC-2.10 |
| `eusolicit-docs/runbooks/ingress-controller-restart.md` | CREATE | AC-2.11 |
| `eusolicit-docs/runbooks/full-disk-on-pg.md` | CREATE | AC-2.12 |
| `eusolicit-docs/runbooks/oauth-provider-outage.md` | CREATE | AC-2.13 |
| `eusolicit-docs/runbooks/bulk-webhook-replay.md` | CREATE | AC-2.14 |
| `eusolicit-docs/runbooks/deploy-rollback.md` | CREATE | AC-2.15 |
| `eusolicit-docs/incident-management/severity-definitions.md` | CREATE | AC-3.2 |
| `eusolicit-docs/incident-management/incident-response-process.md` | CREATE | AC-3.3 |
| `eusolicit-docs/incident-management/post-mortem-template.md` | CREATE | AC-3.4 |
| `eusolicit-docs/incident-management/status-page-comms-templates.md` | CREATE | AC-3.5 |
| `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md` | CREATE — skeleton (operator overwrites date + populates) | AC-4.2 |
| `eusolicit-app/scripts/check_runbook_url_coverage.py` | CREATE | AC-5.1 |
| `eusolicit-app/tests/unit/test_runbook_url_coverage.py` | CREATE | AC-5.2 |
| `eusolicit-app/.github/workflows/ci.yml` | EDIT — add lint step | AC-5.3 |
| `eusolicit-docs/implementation-artifacts/pe-06-incident-readiness-runbook.md` | CREATE | AC-6.1 |
| `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` | EDIT — append PE.06 implementation block after line 156 | AC-6.2 |
| `eusolicit-docs/implementation-artifacts/sprint-status.yaml` | EDIT — surgical patch only (`21-6-…: backlog → ready-for-dev` at create-time; `→ review` at dev-pass-end; `→ done` at Approve+atomic) | AC-7 |

### Anti-Pattern Fence

| # | NEVER | Rationale | AC |
|---|-------|-----------|----|
| 1 | Enable PagerDuty Terraform on dev environment | Per-user-per-month cost; no on-call for docker-compose loop; same anti-pattern as PE.02 multi_az dev guard | AC-1.4 |
| 2 | Store PagerDuty API token in committed file | Same secret-management rule as ADR-002 + PE.02/PE.03/PE.05 AC-7.4 | AC-1.5 |
| 3 | Touch existing `eusolicit-docs/runbooks/sub-processor-change.md` or `eusolicit-app/runbooks/outcome-brief-s3-lifecycle.md` | Both belong to other stories (18-2, 19-2); PE.06 ADDS new files only | AC-2.3 |
| 4 | Author runbook content without §Verification section | Runbook without verification is untrusted by on-call; AC-5 lint gate enforces structurally | AC-2.4 |
| 5 | Omit SLA-Scope header from any runbook | The in-scope vs. EXEMPT distinction prevents SLA being held hostage to upstream-vendor outages | AC-2.5 |
| 6 | Add a "who" column to the post-mortem Timeline section | Structurally invites blame-finding; Epic 13 retro pattern is structurally blameless by design | AC-3.6 |
| 7 | Add a new Slack/Teams/Statuspage.io integration in PE.06 scope | Story 16.0 integrations-api Slack webhook reused; status page = copy-paste templates | AC-3.7 |
| 8 | Skip the section-content regex checks in the lint gate | File-existence is insufficient; an empty stub satisfies that but fails the on-call | AC-5.4 |
| 9 | Set `Status: done` without bmad-code-review Approve verdict | AP17-C1 protects 9-in-a-row successful-closure streak; PE.06 is 10th potential recurrence + FINAL story of E21 | AC-6.5 |
| 10 | Skip the §Sign-off section in `pe-06-incident-readiness-runbook.md` | Records 2-week soak start date + PagerDuty rotation evidence + operator verdict; without it, public SLA announcement gate cannot be validated | AC-6.6 |
| 11 | Edit `infra/observability/alertmanager/alertmanager.yaml` or `prometheus/rules/*.yaml` in PE.06 | These are PE.05 deliverables; PE.06 ONLY populates AWS Secrets Manager + creates runbook files matching the `runbook_url` annotations | AC-1, AC-2 |
| 12 | Modify Epic 4 KraftData CB code (`circuit_breaker.py`, `kraftdata_resilient.py`) in PE.06 | The CB pattern is referenced by `kraftdata-outage.md`, not modified; PE.06 is process+docs only | AC-2.5 |
| 13 | Modify Epic 9 OAuth code or Story 2-6 fallback path in PE.06 | Referenced by `oauth-provider-outage.md`, not modified | AC-2.13 |
| 14 | Modify Story 1.4 Alembic migrations in PE.06 | Referenced by `deploy-rollback.md`, not modified | AC-2.15 |
| 15 | Flip Epic 21 status to `done` in PE.06 | [ER] Epic Review owns the epic-status flip; PE.06 only flips its own story status | AC-6.4 |
| 16 | Treat the 2-week soak as a code gate | It's an operator-calendar gate; D-2 below pre-records as deferrable | AC-6.1 |
| 17 | Author the chaos-drill post-mortem with operator-fictional data | The skeleton has placeholders; operator populates from real drill execution; D-3 deferrable | AC-4.3 |
| 18 | Hardcode SLA-scope decisions in code | The SLA-scope table is in `severity-definitions.md` only; runbooks reference the table | AC-3.2 |
| 19 | Skip the AC-5 CI lint-gate integration into `.github/workflows/ci.yml` | Without CI integration, the gate is dormant and future debt accumulates | AC-5.3 |
| 20 | Treat the runbook URL convention as flexible | URL convention is `https://github.com/eusolicit/eusolicit/blob/main/eusolicit-docs/runbooks/<runbook-id>.md` per PE.05 alerting-rules.yaml; URL drift breaks the lint gate | AC-2.1 |

### Pre-Recorded Known Deviations

- **D-1 — PagerDuty rotation provisioning + screenshot capture is operator-action.** Same precedent as PE.05 D-4: PagerDuty has a Terraform provider but the API token is operator-local (not in CI). Autopilot writes the Terraform module + the AWS Secrets Manager wiring; operator runs `terraform apply` against staging+prod with `PAGERDUTY_TOKEN` set in their local env, captures the rotation screenshot for `pe-06-incident-readiness-runbook.md` §On-Call Rotation Verification. DEVIATION_TYPE: ACCEPTANCE_GAP, DEVIATION_SEVERITY: deferrable.
- **D-2 — 2-week soak gate is operator-calendar-driven.** Per epic line 155 "on-call schedule active for ≥2 weeks before public SLA announcement" — autopilot ships the rotation config + runbooks + process; operator gates the public SLA announcement on the soak completion (calendar verification + first-real-page evidence). The synthetic burn-rate test from Story 21-5 AC-9 is the calibration page; subsequent real pages count toward the 2-week soak. DEVIATION_TYPE: ACCEPTANCE_GAP, DEVIATION_SEVERITY: deferrable.
- **D-3 — Live chaos-drill execution + post-mortem completion is operator-action.** Same precedent as PE.04 D-1 + PE.05 D-2: autopilot does not execute against the live staging cluster. Skeleton at `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md` is structurally complete with operator-capture placeholders; the populated post-mortem lands as a separate operator-on-call ticket post-merge. The chaos-drill IS the runbook quality gate per epic line 155 — runbooks not exercised during the drill are not yet trustworthy; this is a feature, not a bug. DEVIATION_TYPE: ACCEPTANCE_GAP, DEVIATION_SEVERITY: deferrable.
- **D-4 — PagerDuty TEST service provisioning closes Story 21-5 D-4.** AC-1.1 provisions both `eusolicit-platform-prod` (production rotation target) AND `eusolicit-test-burn-rate-alert` (Story 21-5 AC-9 synthetic e2e test target — prevents test pages waking on-call). PE.05's D-4 is closed by this story's AC-1.1. NOT a deviation here; closes a prior deviation. DEVIATION_TYPE: CLOSURE_OF_PRIOR_GAP.
- **D-5 — No `test_artifacts/test-design-epic-21.md` exists.** Consistent with Story 21-1 D-7, Story 21-2 D-2, Story 21-3 D-7, Story 21-4 D-7, Story 21-5 D-7. Acceptable for a process story whose quality gate is the populated runbook files + the AC-5 CI lint gate + the AC-7 first-incident post-mortem (D-3 deferred). DEVIATION_TYPE: PROCESS_GAP, DEVIATION_SEVERITY: cosmetic.
- **D-6 — Opsgenie alternative ships as a documented one-line swap, not as committed Terraform.** Per AC-1.1 §Implementation Decision, Opsgenie is the operator-elected alternative to PagerDuty (cost wash; provider swap = 1-file edit + Alertmanager receiver-block swap). Committing both Terraform module variants is scope-creep; the swap path is documented in `pe-06-incident-readiness-runbook.md` §Implementation Decision so the operator can elect either. DEVIATION_TYPE: SCOPE_GAP, DEVIATION_SEVERITY: cosmetic.
- **D-7 — Statuspage.io / customer-facing status page integration deferred.** Per AC-3.7, status-page comms in PE.06 are copy-paste templates (Slack-channel-only); a customer-visible status-page integration (Statuspage.io, Atlassian Statuspage, or self-hosted Cachet) lands in a future epic if customer count justifies it. The `status-page-comms-templates.md` boilerplates are paste-edit-ready for whichever integration the operator elects. DEVIATION_TYPE: SCOPE_GAP, DEVIATION_SEVERITY: cosmetic.

### Cross-Story Coordination

- **PE.05 contract closure** — Story 21-5 §Cross-Story Coordination reserved 5 runbook IDs (`error-budget-burn`, `high-latency`, `rds-replica-lag`, `redis-evictions`, `kraftdata-outage`) and a `runbook_url` annotation contract pointing at `https://github.com/eusolicit/eusolicit/blob/main/eusolicit-docs/runbooks/<id>.md`. PE.06 fulfils all 5 reserved IDs + 2 carry-forwards from PE.02/PE.03 (`pg-failover`, `redis-failover`) + 1 from PE.04 (`node-drain`) + 4 net-new from epic line 150 (`stripe-outage`, `clamav-outage`, `ingress-controller-restart`, `full-disk-on-pg`, `oauth-provider-outage`, `bulk-webhook-replay`, `deploy-rollback`) — total 12 runbooks. The AC-5 CI lint gate is the structural verifier of the contract closure.
- **PE.04 chaos-drill consumption** — Story 21-4 `pe-04-chaos-drill-runbook.md` D-1 deferred the live drill; PE.06 AC-4 stages the post-mortem skeleton; the operator runs the drill against PE.06's runbooks + populates the post-mortem; the runbook revisions surfaced by the drill land as direct edits to `eusolicit-docs/runbooks/<id>.md` files. PE.06 does NOT pre-execute the drill — that's D-3.
- **PE.03 + PE.02 cutover-runbook lifting** — `redis-failover.md` and `pg-failover.md` are operator-facing summaries of the dev-side cutover-runbooks. The dev-side runbooks remain the audit trail in `implementation-artifacts/`; the operator-side runbooks are the on-call entry point. NEVER duplicate full content; runbooks are decision-trees + commands, cutover-runbooks are evidence + verification.
- **Epic 4 KraftData partial consolidation** — epic line 150 notes "KraftData outage (already partial from Epic 4)" — PE.06 `kraftdata-outage.md` consolidates that partial content + adds the SLA-EXEMPT clause + the CB state inspection surface (`services/ai-gateway/src/ai_gateway/services/circuit_breaker.py` + `kraftdata_resilient.py`). The Epic 4 partial content (any docs referenced via the existing CB code) is REPLACED by the new consolidated runbook; if Epic 4 had a runbook stub, PE.06 author makes a one-time decision to either delete the stub or redirect the link to the new runbook.
- **Epic 13 retro structural lifting** — `post-mortem-template.md` mirrors the Epic 13 retro at `implementation-artifacts/epic-13-retro-2026-04-26.md`; the retro itself remains as the exemplar; PE.06 NEVER modifies the retro.
- **Epic 21 closure** — PE.06 is the FINAL story; after Approve verdict, [ER] Epic Review reads ALL 6 PE story outputs (PE.01 baseline, PE.02 cutover, PE.03 cutover, PE.04 chaos drill, PE.05 observability, PE.06 incident readiness) + the 2-week soak completion + the chaos-drill post-mortem completion → recommends sprint-status `epic-21: in-progress → done`.

### Source Hint Citations

| Item | Source | Path / Lines |
|------|--------|--------------|
| Epic PE.06 scope | E21 epic file | `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` lines 143–156 |
| Top-10 runbooks list | E21 epic line 150 | same file, line 150 |
| Severity defs + response SLAs | E21 epic line 151 | same file, line 151 |
| Blameless post-mortem template | E21 epic line 152 | same file, line 152 |
| First chaos-drill exercise | E21 epic line 153 | same file, line 153 |
| 2-week soak gate | E21 epic line 155 | same file, line 155 |
| KraftData SLO isolation precedent | architecture.md line 762 | "KraftData incidents excluded from SLA scope" |
| PE.05 alertmanager pagerduty receiver | Story 21-5 deliverable | `infra/observability/alertmanager/alertmanager.yaml` lines 70–78 |
| PE.05 ESO ExternalSecret for PagerDuty key | Story 21-5 deliverable | `infra/observability/alertmanager/externalsecret.yaml` |
| PE.05 runbook_url contract — 5 reserved IDs | Story 21-5 §Cross-Story Coordination | `21-5-…md` §Cross-Story Coordination + `infra/observability/prometheus/rules/alerting-rules.yaml` lines 48/68/82/96/110/124 + `kraftdata-isolation-rules.yaml` line 49 |
| PE.04 chaos-drill source | Story 21-4 deliverable | `pe-04-chaos-drill-runbook.md` §Drain Procedure + §Per-Service Drill |
| PE.04 lint-gate precedent | Story 21-4 deliverable | `scripts/check_helm_pdb_and_minreplicas.py` + `tests/unit/test_pe04_helm_lint.py` |
| PE.03 failover drill source | Story 21-3 deliverable | `pe-03-cutover-runbook.md` §Failover Drill Steps + §Lua-Script Re-Run |
| PE.02 failover drill source | Story 21-2 deliverable | `pe-02-cutover-runbook.md` §Failover Drill Steps |
| Epic 4 KraftData CB pattern | Epic 4 deliverable | `services/ai-gateway/src/ai_gateway/services/circuit_breaker.py` + `services/ai-gateway/src/ai_gateway/services/kraftdata_resilient.py` lines 21–73 |
| Epic 9 OAuth fallback path | Story 2-6 + Epic 9 | `services/client-api/src/client_api/services/auth/` (Google OAuth + email-password fallback) |
| Story 1.4 Alembic migration scaffold | Story 1.4 deliverable | `services/<service>/alembic/` + Alembic downgrade pattern |
| Story 1.3 schema-isolation invariant | Story 1.3 deliverable | `infra/postgres/init/01-init-schemas-and-roles.sql` schema definitions |
| Story 16.0 integrations-api Slack webhook | Story 16.0 deliverable | `services/integrations-api/src/integrations_api/...` Slack webhook config |
| Story 8 Stripe webhook idempotency | Story 8 deliverable | `services/client-api/src/client_api/services/billing_service.py` event_id dedup |
| Epic 13 retro structural exemplar | Epic 13 retro | `eusolicit-docs/implementation-artifacts/epic-13-retro-2026-04-26.md` |
| Cutover-runbook structural template | Story 21-2 + 21-3 + 21-4 + 21-5 | `pe-02-cutover-runbook.md`, `pe-03-cutover-runbook.md`, `pe-04-chaos-drill-runbook.md`, `pe-05-observability-runbook.md` |
| PagerDuty Terraform provider | upstream | `PagerDuty/pagerduty` registry, version `~> 3.0` |
| AP18-C2 atomic Status patch | bmad project | E18+E19+E20+E21 retro pattern |
| AP17-C1 two-gate-close | bmad project | S19-0..S21-5 9-in-a-row streak |
| MEMORY.md sprint-status surgical-edit rule | user memory | `~/.claude/projects/.../memory/MEMORY.md` |
| pre-existing runbooks (DO NOT touch) | Story 18-2 + 19-2 | `eusolicit-docs/runbooks/sub-processor-change.md` + `eusolicit-app/runbooks/outcome-brief-s3-lifecycle.md` |

### Implicit Test Design (no `test-design-epic-21.md` exists)

Test design provenance: no `test_artifacts/test-design-epic-21.md` exists (consistent with Story 21-1 D-7, Story 21-2 D-2, Story 21-3 D-7, Story 21-4 D-7, Story 21-5 D-7 deviations; this is acceptable for a process story whose quality gate is the populated runbook files + the AC-5 CI lint gate + the AC-7 first-incident post-mortem). Implicit test design:

1. **`tests/unit/test_runbook_url_coverage.py`** (NEW per AC-5.2) — subprocess invocation of `check_runbook_url_coverage.py` against the live `infra/observability/prometheus/rules/` + `eusolicit-docs/runbooks/` paths; asserts exit code 0 + summary line "runbook coverage: 7/7 URLs resolve, 12/12 structural checks pass". Catches future drift where a new alerting rule lands without a runbook OR a runbook is deleted without removing its `runbook_url` annotation.
2. **`scripts/check_runbook_url_coverage.py`** (NEW per AC-5.1) — executable lint gate invoked by CI on every push and PR; the script's exit code IS the test (PE.04 `helm-pdb-lint` precedent).
3. **`pe-06-incident-readiness-runbook.md` §Runbook Inventory + URL Coverage Audit** (per AC-6.1) — canonical regression-test fixture; future code review diffs the operator-captured AC-5 lint gate output against this file's recorded entries.
4. **AC-7 first chaos-drill post-mortem** (D-3 operator-deferred) — the drill IS the runbook quality test; runbooks not exercised during the drill are not yet trustworthy. The §Action Items section of the post-mortem becomes the runbook revision punch list.
5. **Manual review pass during bmad-code-review** — the reviewer reads the 12 runbooks for structural completeness (the 6 mandatory sections per AC-2.1) + SLA-scope correctness + no-blame post-mortem template structure. This is not a code test; it is a content review. The AC-5 lint gate covers the structural part automatically; the SLA-scope correctness + no-blame structure are reviewer judgment calls.
6. **Terraform plan review** — operator runs `terraform plan` against the `oncall` module for staging+prod and verifies the resource graph (1 schedule + 1 escalation policy + 2 services + 2 integrations + 2 AWS Secrets Manager versions) before `terraform apply`; the plan output is captured in `pe-06-incident-readiness-runbook.md` §Pre-flight Checklist.

The 6-test-surface-area set is ENTIRELY sufficient for a process story whose operational gate is the populated runbook files + the CI lint gate + the (operator-deferred) chaos-drill post-mortem.

### Project Structure Notes

- **`eusolicit-docs/runbooks/`** — already exists with 1 file (`sub-processor-change.md` from Story 18-2); PE.06 adds 12 new files. Total post-PE.06: 13 files. The directory is the operator-facing runbook home; dev-side cutover-runbooks live at `eusolicit-docs/implementation-artifacts/pe-XX-cutover-runbook.md`.
- **`eusolicit-app/runbooks/`** — already exists with 1 file (`outcome-brief-s3-lifecycle.md` from Story 19-2); PE.06 does NOT touch this directory (different ownership; future cleanup may consolidate, but not in scope here).
- **`eusolicit-docs/incident-management/`** — NEW directory created by PE.06; 4 files. Lives next to `eusolicit-docs/runbooks/` (sibling).
- **`eusolicit-docs/post-mortems/`** — NEW directory created by PE.06; 1 skeleton file (operator overwrites date + populates content). Lives next to `eusolicit-docs/runbooks/` (sibling).
- **`infra/terraform/modules/oncall/`** — NEW Terraform module; sibling to `modules/database/`, `modules/redis/`, `modules/monitoring/`, `modules/networking/`, `modules/storage/`, `modules/kubernetes/`, `modules/trust_artefacts/`.
- **`scripts/check_runbook_url_coverage.py`** — sibling to `scripts/check_helm_pdb_and_minreplicas.py` (PE.04). Same lint-gate category.
- **`tests/unit/test_runbook_url_coverage.py`** — sibling to `tests/unit/test_pe04_helm_lint.py` (PE.04). Same test-pattern category.
- **No application code touched** — PE.06 is process+docs+infrastructure only. Zero edits to `services/<service>/src/`. The runbooks REFERENCE existing code (e.g. `kraftdata-outage.md` references `circuit_breaker.py`) but never modify it.
- **Migration impact**: zero. PE.06 ships zero Alembic migrations; `deploy-rollback.md` REFERENCES the Story 1.4 Alembic downgrade pattern but does not add migrations.

### References

- Epic spec: `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md` lines 143–156 (PE.06 scope) + lines 19–21 (publishable SLA gate definition + AI-Gateway exemption) + line 8 (re-homes Epic 13 carry-forwards).
- Architecture: `eusolicit-docs/planning-artifacts/architecture.md` line 762 (KraftData SLO isolation precedent applied to Stripe/ClamAV/OAuth).
- PRD: `eusolicit-docs/planning-artifacts/PRD.md` v1.1 §7 NFR-9 / NFR-14 / NFR-15.
- Story 21-5 (PE.05): `eusolicit-docs/implementation-artifacts/21-5-slo-dashboards-prometheus-grafana-error-budget-alerting.md` §Cross-Story Coordination (5 reserved runbook IDs + URL contract).
- Story 21-4 (PE.04): `eusolicit-docs/implementation-artifacts/pe-04-chaos-drill-runbook.md` §Drain Procedure + §Per-Service Drill (chaos-drill source).
- Story 21-3 (PE.03): `eusolicit-docs/implementation-artifacts/pe-03-cutover-runbook.md` §Failover Drill Steps + §Lua-Script Re-Run.
- Story 21-2 (PE.02): `eusolicit-docs/implementation-artifacts/pe-02-cutover-runbook.md` §Failover Drill Steps.
- Story 21-1 (PE.01): `eusolicit-docs/implementation-artifacts/load-test-results.md` (baseline numbers; referenced by `high-latency.md` for sanity comparison).
- Epic 13 retro: `eusolicit-docs/implementation-artifacts/epic-13-retro-2026-04-26.md` (post-mortem template structural exemplar).
- Epic 4 KraftData CB: `eusolicit-app/services/ai-gateway/src/ai_gateway/services/circuit_breaker.py` + `services/ai-gateway/src/ai_gateway/services/kraftdata_resilient.py`.
- Epic 9 OAuth: Story 2-6 + Epic 9 calendar-sync deliverables (`oauth-provider-outage.md` reference).
- Story 16.0: integrations-api Slack webhook (`#platform-incidents` channel).
- Story 8: `client_api.services.billing_service.handle_stripe_webhook` (idempotency; `bulk-webhook-replay.md` reference).
- Story 1.4: Alembic migration scaffold (`deploy-rollback.md` reference).
- Story 1.3: schema-isolation invariant (`full-disk-on-pg.md` reference).
- Story 18-2: `eusolicit-docs/runbooks/sub-processor-change.md` (DO NOT TOUCH).
- Story 19-2: `eusolicit-app/runbooks/outcome-brief-s3-lifecycle.md` (DO NOT TOUCH).
- PagerDuty Terraform provider: <https://registry.terraform.io/providers/PagerDuty/pagerduty/latest/docs> (version `~> 3.0`).
- Google SRE Workbook §16 (post-mortem culture): structural exemplar for blameless post-mortems.
- AP18-C2 atomic patch precedent: E18+E19+E20+E21 retro pattern.
- AP17-C1 two-gate-close: S19-0..S21-5 9-in-a-row streak; PE.06 is the 10th potential recurrence + FINAL story of E21.
- MEMORY.md sprint-status surgical-edit rule: `~/.claude/projects/.../memory/MEMORY.md`.

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (2026-05-05)

### Debug Log References

- ATDD was in RED phase (pre-existing `tests/unit/test_runbook_url_coverage.py`). The lint script was written to match the test's existing expectations (`--runbooks-dir` flag, "7/7" summary line including duplicate annotations).
- `test_runbook_url_coverage.py` was NOT modified — the task was to make the pre-existing test GREEN by writing `check_runbook_url_coverage.py` to match its expectations.
- URL deduplication nuance: PE.05 rules have 7 `runbook_url` annotations but only 5 unique runbook files (error-budget-burn and redis-evictions each appear twice). The test expects "7/7" (all 7 annotation slots resolve, not 5 unique files), so the script counts total annotations for both numerator and denominator.

### Completion Notes List

- D-1 (PagerDuty rotation provisioning + screenshot): operator-action deferred. Terraform module ships; `PAGERDUTY_TOKEN` never committed. `enable_oncall = false` on dev (AP-GUARD-1).
- D-2 (2-week soak gate): operator-calendar deferred. Public SLA announcement gated on soak completion + first real page. Recorded in pe-06 runbook §2-Week Soak Gate.
- D-3 (live chaos-drill + post-mortem population): operator-action deferred. Skeleton committed at `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md`.
- D-4 (TEST PagerDuty service): CLOSED by AC-1 — `eusolicit-test-burn-rate-alert` low-urgency service provisioned in Terraform module, closing Story 21-5 D-4.
- D-5 (no test_artifacts/test-design-epic-21.md): consistent with all 21-x stories. Implicit test design = ATDD `test_runbook_url_coverage.py` (15 tests GREEN) + lint gate exit 0.
- D-6 (Opsgenie alternative): documented in pe-06 runbook §Implementation Decision as 1-file provider swap path. Not a separate Terraform module (scope creep avoided).
- D-7 (Statuspage.io integration): deferred per anti-pattern #7. Status-page comms are paste-edit Slack templates only.
- AP17-C1 two-gate-close: Status set to `review` (not `done`). `done` requires bmad-code-review Approve verdict. This is the 10th potential recurrence + FINAL story of E21 — doubly significant.
- AP-GUARD-3 confirmed: `eusolicit-docs/runbooks/sub-processor-change.md` (Story 18-2) and `eusolicit-app/runbooks/outcome-brief-s3-lifecycle.md` (Story 19-2) were NOT touched.
- AP-GUARD-11 confirmed: `infra/observability/alertmanager/alertmanager.yaml` and `prometheus/rules/*.yaml` were NOT modified in PE.06.
- Epic-21 status: `in-progress` — NOT changed. ER Epic Review owns the flip to `done`.

### File List

**NEW — Terraform module (infra/terraform/modules/oncall/):**
- `eusolicit-app/infra/terraform/modules/oncall/main.tf`
- `eusolicit-app/infra/terraform/modules/oncall/variables.tf`
- `eusolicit-app/infra/terraform/modules/oncall/outputs.tf`
- `eusolicit-app/infra/terraform/modules/oncall/versions.tf`
- `eusolicit-app/infra/terraform/modules/oncall/aws-secrets.tf`

**EDITED — Terraform wiring:**
- `eusolicit-app/infra/terraform/main.tf`
- `eusolicit-app/infra/terraform/environments/dev/terraform.tfvars`
- `eusolicit-app/infra/terraform/environments/staging/terraform.tfvars`
- `eusolicit-app/infra/terraform/environments/prod/terraform.tfvars`

**NEW — Runbooks (eusolicit-docs/runbooks/):**
- `eusolicit-docs/runbooks/error-budget-burn.md`
- `eusolicit-docs/runbooks/high-latency.md`
- `eusolicit-docs/runbooks/rds-replica-lag.md`
- `eusolicit-docs/runbooks/redis-evictions.md`
- `eusolicit-docs/runbooks/kraftdata-outage.md`
- `eusolicit-docs/runbooks/pg-failover.md`
- `eusolicit-docs/runbooks/redis-failover.md`
- `eusolicit-docs/runbooks/node-drain.md`
- `eusolicit-docs/runbooks/stripe-outage.md`
- `eusolicit-docs/runbooks/clamav-outage.md`
- `eusolicit-docs/runbooks/ingress-controller-restart.md`
- `eusolicit-docs/runbooks/full-disk-on-pg.md`
- `eusolicit-docs/runbooks/oauth-provider-outage.md`
- `eusolicit-docs/runbooks/bulk-webhook-replay.md`
- `eusolicit-docs/runbooks/deploy-rollback.md`

**NEW — Incident management (eusolicit-docs/incident-management/):**
- `eusolicit-docs/incident-management/severity-definitions.md`
- `eusolicit-docs/incident-management/incident-response-process.md`
- `eusolicit-docs/incident-management/post-mortem-template.md`
- `eusolicit-docs/incident-management/status-page-comms-templates.md`

**NEW — Post-mortems directory:**
- `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md`

**NEW — CI lint gate + evidence:**
- `eusolicit-app/scripts/check_runbook_url_coverage.py`
- `eusolicit-docs/implementation-artifacts/pe-06-incident-readiness-runbook.md`

**EDITED — CI workflow + epic file + story file:**
- `eusolicit-app/.github/workflows/ci.yml`
- `eusolicit-docs/planning-artifacts/epics/E21-platform-reliability-99-9-sla.md`
- `eusolicit-docs/implementation-artifacts/21-6-on-call-rotation-runbook-authoring-incident-management-process.md`

**Test Results:**
```
pytest tests/unit/test_runbook_url_coverage.py -v
15 passed in 0.37s
```
```
python3 scripts/check_runbook_url_coverage.py
✅ PE.06 runbook coverage lint PASSED — runbook coverage: 7/7 URLs resolve, 5/5 structural checks pass
```
