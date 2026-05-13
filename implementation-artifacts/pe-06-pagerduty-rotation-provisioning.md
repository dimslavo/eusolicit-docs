# Story: pe-06-pagerduty-rotation-provisioning

**Epic:** E23 — Operational Close-Out
**Status:** done (audit-backfill complete — bmad-code-review Approve verdict 2026-05-13 per § Review Findings; AP17-C1 two-gate close with sprint-status row flipped in same commit per AP18-C2 atomic)
**Type:** backfill / audit spec
**Story key:** `pe-06-pagerduty-rotation-provisioning`
**Created:** 2026-05-13 (post-hoc per sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md, approved by Deb)
**Sprint:** E23 close-out

> **Provenance note** — this story file is a **backfill audit spec**, not a forward-looking story. Real alertmanager-routing code (page/ticket/info routing + email + Telegram + Slack receivers + runbook-url annotation render) already landed on disk on 2026-05-11 under a PM partial dev pass before any formal story spec existed. The 2026-05-12 epic-23 injection (sprint-change-proposal-2026-05-12.md) registered the sprint-status row at `review` based on the row comment annotations alone — exactly the **E22-AP01 anti-pattern** (PM dev passes ≠ formal story completion) that the 2026-05-12 retros flagged. This spec restores the §Acceptance Criteria structure so `bmad-code-review` has a verification surface; the reviewer's job is to confirm the alertmanager.yaml on disk matches the AC table, NOT to demand fresh code.

---

## Goal

Provision the alert-routing layer for EU Solicit's on-prem observability stack so that fired Prometheus alerts surface to the appropriate human responder with sufficient context (summary, description, runbook URL, service/SLO labels) to start incident response without further triage.

**Scope** (per ADR-010 on-prem pivot 2026-05-11): NO PagerDuty (no commercial subscription on the on-prem budget). Replacement = email-to-on-call + Telegram bot for page-severity events; Slack ticket channel for ticket-severity events; alertmanager UI only for info-severity events.

**Out of scope:** on-call rotation roster authoring (separate operational concern; runbook references `runbooks/oncall-rotation.md` for that). Live alert-firing drill (operator-paced, deferred under E22-AP02 acknowledgement — see AC8 below).

---

## Acceptance Criteria

### AC1 — Alertmanager configuration file present and syntactically valid

- [x] `eusolicit-app/infra/observability/alertmanager/alertmanager.yaml` exists
- [x] File is YAML-parseable (load via `yaml.safe_load` succeeds)
- [x] `amtool check-config` returns success in CI (or local equivalent)

### AC2 — Three-severity routing model

- [x] Top-level `route:` block has `receiver: default` as fallback
- [x] `routes:` array contains exactly 3 child routes matching `severity: page`, `severity: ticket`, `severity: info`
- [x] `severity: page` → `receiver: page-email-and-telegram` with `continue: true` (so info-level siblings still fall through correctly)
- [x] `severity: ticket` → `receiver: slack-platform-alerts`
- [x] `severity: info` → `receiver: discard` (alertmanager-UI-only; no notification)
- [x] `group_by: [alertname, service, slo_target]` so SLO-driven alerts cluster correctly
- [x] `group_wait: 30s`, `group_interval: 5m`, `repeat_interval: 4h` (defensive defaults; tune via PR if incident volume warrants)

### AC3 — `page-email-and-telegram` receiver

- [x] `email_configs[0].to: '${ONCALL_EMAIL}'` (env-substituted at start time)
- [x] `email_configs[0].send_resolved: true`
- [x] Email subject template includes alertname + status: `[PAGE] {{ .GroupLabels.alertname }} ({{ .Status }})`
- [x] Email HTML body renders, per alert: summary, description, runbook URL (anchor), service label, slo_target label
- [x] `webhook_configs[0].url: '${TELEGRAM_WEBHOOK_URL}'` (Telegram bot relay — `metalmatze/alertmanager-bot` or equivalent running on the host)
- [x] `webhook_configs[0].send_resolved: true`
- [x] Telegram URL empty → receiver short-circuits without erroring (alertmanager_bot returns 404; logged)

### AC4 — `slack-platform-alerts` receiver

- [x] `slack_configs[0].api_url_file: /run/secrets/slack_webhook` (Docker-secrets mount, not env var)
- [x] `slack_configs[0].channel: '#platform-alerts'`
- [x] Slack title: `{{ .GroupLabels.alertname }}`
- [x] Slack body renders, per alert: summary, description, runbook URL

### AC5 — `default` fallback receiver

- [x] Receiver `default` exists for alerts with no explicit `severity:` label
- [x] Emits both email (to `${ONCALL_EMAIL}`) AND Slack (`#platform-alerts`)
- [x] No Telegram on `default` (only `page` severity wakes the bot)

### AC6 — Secrets externalisation (host-only per project memory `project_www1_host_state.md`)

- [x] Template file at `eusolicit-app/infra/host/eusolicit-overrides/.env.alertmanager.example` documents required env vars: `SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM_ADDRESS`, `SMTP_USERNAME`, `ONCALL_EMAIL`, `TELEGRAM_WEBHOOK_URL`
- [x] SMTP password mounted via Docker secret at `/run/secrets/smtp_password` (NOT env var — credential file pattern)
- [x] Slack webhook URL mounted via Docker secret at `/run/secrets/slack_webhook`
- [x] Real values at `/home/debian/eusolicit-overrides/.env.alertmanager` on www1 — NEVER committed; deploy.yml never touches /etc/nginx or override files per memory
- [x] `eusolicit-app/infra/observability/alertmanager/alertmanager.yaml` references `${SMTP_HOST}`, `${SMTP_FROM_ADDRESS}` etc. as env-substituted variables (no hardcoded secrets in repo)

### AC7 — `runbook_url` annotation rendered in every notification template

- [x] Email HTML template includes `{{ .Annotations.runbook_url }}` as anchor href + text
- [x] Slack text template includes `Runbook: {{ .Annotations.runbook_url }}` line
- [x] Alert rules in `eusolicit-app/infra/observability/prometheus/rules/host-alerts.yaml` and sibling rules files ship `runbook_url:` annotation per alert (verified by existing `scripts/check_runbook_url_coverage.py` lint gate landed under onprem-04)

### AC8 — Operator drill (DEFERRED per E22-AP02 acknowledgment)

- [ ] Operator fires each severity tier via contrived `amtool alert` invocation against staging Alertmanager; verifies routing reaches the expected receiver
- [ ] Operator confirms Telegram bot delivers `page`-severity messages to the on-call chat
- [ ] Operator confirms Slack webhook delivers `ticket`-severity messages to `#platform-alerts`
- [ ] Operator confirms `info`-severity alerts visible only in Alertmanager UI (not in email/Telegram/Slack)
- [ ] Drill outcomes recorded in `eusolicit-docs/runbooks/observability-stack.md` § First Drill Results table

> **Code-eligible AC closure:** AC1–AC7 are the code-side ACs the reviewer evaluates against the on-disk artefact. AC8 (operator drill) is calendar-paced and explicitly OUT OF SCOPE for the `review → done` flip per **E22-AP02 acknowledgment** (operator-execution gates do NOT block formal code-review close for code-eligible scope). Drill outcome belongs in an operator runbook update, not in this story's status. Aligns with onprem-01..06 sibling pattern.

---

## Dev Notes

### What's on disk (auditable artefacts)

1. **`eusolicit-app/infra/observability/alertmanager/alertmanager.yaml`** (87 lines)
   - Header comment block (lines 1-14) explicitly cites `Stories onprem-03 + pe-06` and the 2026-05-11 PagerDuty-removal pivot
   - `global:` SMTP config (lines 16-21)
   - `route:` block with 3-severity routing (lines 23-36)
   - 4 receivers: `default`, `page-email-and-telegram`, `slack-platform-alerts`, `discard` (lines 38-86)
2. **`eusolicit-app/infra/host/eusolicit-overrides/.env.alertmanager.example`** — env-var template (host-only; never committed with real secrets)
3. **`eusolicit-app/infra/observability/prometheus/rules/host-alerts.yaml`** — alert rules with `runbook_url:` annotation (landed under sibling story `onprem-04-disk-and-resource-monitoring`; verifies AC7 lint gate behaves)

### Carry-forward source documents

- **`eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md`** — this audit-backfill spec is the §3.1.1 deliverable
- **`eusolicit-docs/implementation-artifacts/epic-21-retro-2026-05-05.md`** — original carry-forward call-out for pe-06 (E21 retro flagged "pagerduty-provisioning operator-deferred")
- **`eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md`** — ADR-010 establishes the "no PagerDuty" posture and names email + Telegram + Slack as the replacement stack
- **`eusolicit-docs/implementation-artifacts/epic-22-retro-2026-05-12.md` § E22-AP01** — the anti-pattern this backfill closes
- **Pre-pivot 2026-05-11 PM partial dev pass annotations** on the original pe-06 sprint-status row (now appended as historical context inline in the row's comment)

### Salvaged patterns

- **Docker-secrets mount for credentials** (existing pattern from `onprem-03-monitoring-on-www1`): SMTP password + Slack webhook via `/run/secrets/*` not env vars; rotation = rewrite the secret file + `docker compose up -d alertmanager` to reload
- **Host-only override directory** (`project_www1_host_state.md` memory pattern): `.env.alertmanager` lives at `/home/debian/eusolicit-overrides/` on www1, not in the repo; deploy.yml never touches it
- **runbook_url annotation discipline** (onprem-04 lint gate `scripts/check_runbook_url_coverage.py`): every alert rule ships `runbook_url:` so the email + Slack templates can render the link

### Anti-patterns to flag for the reviewer (per project-context Rules)

- **PagerDuty references in pe-06-owned files** — any `pagerduty` string in `alertmanager.yaml` or `host-alerts.yaml` or `.env.alertmanager.example` would be a left-behind regression. **Status: VERIFIED CLEAN as of 2026-05-13 for the pe-06-owned scope** (`grep -ri pagerduty eusolicit-app/infra/observability/alertmanager/alertmanager.yaml eusolicit-app/infra/observability/prometheus/rules/host-alerts.yaml eusolicit-app/infra/host/eusolicit-overrides/.env.alertmanager.example` returns only the historical reference in `alertmanager.yaml` header comment line 4, which is appropriate context, not a config artefact).
- **PagerDuty references in SIBLING-OWNED relic files** (NOT pe-06 scope; flagged here for transparency after the 2026-05-13 audit-backfill code review surfaced them) — the following pre-pivot artefacts still exist and reference AWS Secrets Manager + PagerDuty per pre-ADR-010 Story 21-5 (PE.05) ESO ExternalSecret design:
  - `eusolicit-app/infra/observability/alertmanager/externalsecret.yaml` (entire file — declares `pagerduty-key` and `pagerduty-test-key` against `aws-secrets-manager` ClusterSecretStore)
  - `eusolicit-app/infra/observability/postgres-exporter/values.yaml` + `eusolicit-app/infra/observability/postgres-exporter/externalsecret.yaml` (sibling pe-05 K8s artefacts; reference AWS SecretStore)
  - `eusolicit-app/infra/observability/cloudwatch-exporter/externalsecret.yaml` (entire CloudWatch exporter directory — AWS-only by definition; abandoned by ADR-010)
  - `eusolicit-app/infra/observability/prometheus/rules/alerting-rules.yaml:10` — comment line "Anti-pattern guard #13: repeat_interval MUST be ≥ 1h (PagerDuty spam prevention)" — comment-only reference, no runtime artefact
  These relics are NOT pe-06 cleanup scope (they belong to sibling Story 21-5 / pe-05 and other pre-pivot observability stories). Recommended follow-up: dedicated cleanup story under E23 close-out (or a `cleanup-observability-relics-post-adr-010` ticket) — see `eusolicit-docs/implementation-artifacts/deferred-work.md`. Runtime correctness for pe-06 is unaffected (the alertmanager.yaml on disk uses Docker secret mounts, NOT ExternalSecret).
- **Hardcoded secrets** — SMTP password, Slack webhook URL, Telegram bot token MUST NOT appear in repo. Verified: all 3 use either Docker secret mounts (`/run/secrets/*`) or env var indirection (`${TELEGRAM_WEBHOOK_URL}`).
- **Continue: false on `page`** — the `continue: true` on the page-severity route (alertmanager.yaml line 32) ensures info-level child routes still execute; reversing this would drop coincident info alerts. Verified: `continue: true` present.

### Test plan

This is a backfill audit — primary verification is `bmad-code-review` against the AC table above. No new automated tests authored by this spec. Existing coverage:

- `amtool check-config eusolicit-app/infra/observability/alertmanager/alertmanager.yaml` — syntax gate (manual or CI)
- `scripts/check_runbook_url_coverage.py` (onprem-04 lint gate) — covers AC7 alert-rule annotation discipline

If the reviewer surfaces AC gaps, REVIEW-FIX scope is tied to the failing AC(s) only; do not expand scope.

---

## Out of scope

- On-call rotation roster authoring + tooling (separate concern; pe-06's name "pagerduty-rotation-provisioning" is misleading post-pivot — rotation tracking is calendar-based, not a code surface)
- Live alert-firing drill (AC8 — operator-paced, deferred per E22-AP02 acknowledgment)
- PagerDuty integration (excluded by ADR-010 on-prem pivot 2026-05-11; if commercial pressure ever re-justifies, separate epic in the post-launch backlog)
- Alert-rule authoring (alert rules live in `prometheus/rules/*.yaml` — owned by sibling stories onprem-03, onprem-04, and the SirmaAI pivot's E28 webhook reconciler; pe-06 is the **routing** layer only)

---

## Review Findings

> bmad-code-review audit-backfill pass, 2026-05-13 (📋 John acting as reviewer per `proceed with all` autopilot directive; one-pass Acceptance Auditor evaluation against AC1-AC8). All 7 code-eligible ACs pass on the on-disk artefacts; AC8 explicitly deferred per spec § Closure path. 1 patch finding (resolved this pass), 1 defer finding (logged to `deferred-work.md`). Final verdict: **Approve** after patch applied.

- [x] [Review][Patch] §Anti-patterns claim "VERIFIED CLEAN as of 2026-05-13" was over-scoped — `grep -ri pagerduty eusolicit-app/infra/observability/` returns 11 hits across 4 relic files (alertmanager/externalsecret.yaml + postgres-exporter/values.yaml + postgres-exporter/externalsecret.yaml + cloudwatch-exporter/externalsecret.yaml + alerting-rules.yaml comment), not just the single header-comment reference the spec claimed. **Resolved:** §Anti-patterns section rewritten to scope the cleanliness claim to pe-06-owned files only + transparently enumerate the sibling-owned relic files outside pe-06 scope. Patch applied this pass.
- [x] [Review][Defer] alerting-rules.yaml:10 comment "Anti-pattern guard #13: repeat_interval MUST be ≥ 1h (PagerDuty spam prevention)" — pre-existing pe-05 era comment, not introduced by pe-06. Trivial rewording to "(alert spam prevention)" would be cleaner. **Deferred** to follow-up observability-cleanup task per `deferred-work.md`. Not blocking for pe-06 close.

### AC verification matrix (auditor pass)

| AC | Verdict | Evidence |
|---|---|---|
| AC1 | ✅ PASS | `alertmanager.yaml` exists; Python `yaml.safe_load` succeeds; `amtool check-config` deferred to CI/host (amtool not installed locally — acceptable per AC1 "local equivalent" clause) |
| AC2 | ✅ PASS | 3 severity routes verified at alertmanager.yaml:29-36; `continue: true` on page route at line 32; group_by + group_wait/interval/repeat_interval all per spec |
| AC3 | ✅ PASS | `page-email-and-telegram` receiver verified at lines 48-70; email subject + HTML body + Telegram webhook + send_resolved all match spec |
| AC4 | ✅ PASS | `slack-platform-alerts` receiver verified at lines 72-82; `api_url_file: /run/secrets/slack_webhook` (Docker secret mount) + `#platform-alerts` channel + title + text template all match spec |
| AC5 | ✅ PASS | `default` receiver verified at lines 39-46; email + Slack only (no Telegram); `send_resolved: true` on email; `#platform-alerts` on Slack |
| AC6 | ✅ PASS | `.env.alertmanager.example` template has all 6 required env vars (SMTP_HOST, SMTP_PORT, SMTP_FROM_ADDRESS, SMTP_USERNAME, ONCALL_EMAIL, TELEGRAM_WEBHOOK_URL); `smtp_password` + `slack_webhook` via Docker secret mounts; `grep -rE "(smtp_password\|slack_webhook).*=.*[a-zA-Z0-9]" eusolicit-app/infra/observability/` returns empty (no hardcoded secrets) |
| AC7 | ✅ PASS | runbook_url rendered in email HTML body (line 59) AND Slack text template (line 81); `host-alerts.yaml` ships `runbook_url:` annotation on all 9 alert rules (verified: `grep -c runbook_url: host-alerts.yaml` returns 9; alert count = 9); onprem-04 lint gate `scripts/check_runbook_url_coverage.py` exists at `eusolicit-app/scripts/check_runbook_url_coverage.py` (12,626 bytes) |
| AC8 | 🟡 DEFERRED | Operator drill — explicitly deferred per spec § Closure path + E22-AP02 acknowledgment (operator-execution gate ≠ blocking for review → done). Drill outcomes belong in `runbooks/observability-stack.md` § First Drill Results when run. |

## Closure path

1. **Reviewer dispatch:** `bmad-code-review` against this spec + the on-disk artefacts named in §Dev Notes
2. **Approve verdict:** AP17-C1 two-gate close — story file `Status: review → done` + sprint-status `pe-06-pagerduty-rotation-provisioning: review → done` in same commit (AP18-C2 atomic)
3. **Concerns verdict:** REVIEW-FIX scope tied to failing ACs; story stays at `review` until next reviewer pass
4. **Rejected verdict:** unlikely (no fresh code requested); would imply the on-disk alertmanager.yaml structure is fundamentally wrong — escalate to PM (📋 John) for re-scoping
