---
Status: backlog
Epic: E22 — On-Prem Launch
Story type: platform-engineering + ops
Owner: platform-engineering
Created: 2026-05-15 (via IR remediation — `implementation-readiness-report-2026-05-15.md` Issue #11)
Source: PRD amendment 2026-05-12 §SaaS B2B Integrations (Data Ingestion); architecture amendment §ADR-018 (shared N8N at Org level); MEMORY rule [Auto-sync ships unverified code]
---

# onprem-07-n8n-instance-ops

## Goal

Codify operational ownership of the shared EU Solicit-owned N8N instance on `www1.endigitalx.com`: backup, restore, upgrade, version pinning, observability, and incident runbook. E05 amendment owns the *workflow templates* that run inside N8N; this story owns the *N8N container itself*. Without this story the N8N substrate has no upgrade/backup/SLO owner — a class of risk equivalent to the auto-sync defect class flagged in `project_auto_sync_quality.md`.

## Acceptance Criteria

- [ ] **AC1** — N8N container runs as part of the `docker-compose.prod.yml` stack on www1 with explicit version pinning (image tag = specific N8N release, NOT `latest`); upgrade policy documented (manual, monthly review of N8N release notes; major-version bumps require a sprint-change-proposal and runbook).
- [ ] **AC2** — N8N data persisted to a host-mounted volume `/home/debian/eusolicit-overrides/n8n-data/` with file-mode 700; volume included in the daily backup set (per `onprem-01` Hetzner Storage Box replication). Backup contents verified: N8N workflows JSON export + credentials store + execution history.
- [ ] **AC3** — Weekly automated workflow export (`n8n export:workflow --all --output=…`) committed to repo at `infra/n8n/workflows-export/` (idempotent — re-export should produce stable diffs). Doubles as a versioned record of which workflow templates are currently live.
- [ ] **AC4** — Restore runbook at `eusolicit-docs/runbooks/n8n-restore.md` covering: (a) fresh-container bootstrap, (b) volume restore from Hetzner Storage Box, (c) workflow re-import from `infra/n8n/workflows-export/` if volume restore fails, (d) credential re-bind for SirmaAI access tokens (Fernet-encrypted in EU Solicit Postgres per E04 amendment) + N8N webhook secrets, (e) post-restore validation (run a test workflow end-to-end to verify SirmaAI connectivity).
- [ ] **AC5** — Prometheus scraping enabled on N8N's `/metrics` endpoint (N8N >= 1.34 supports it natively); Grafana dashboard `infra/observability/grafana/dashboards/n8n-overview.json` with 4 panels: workflow-execution success rate (by template), p95 execution duration, queue depth, container uptime.
- [ ] **AC6** — Alertmanager alerts (via `onprem-03` alertmanager.yaml): N8N container down >1min (page); workflow-execution success rate <90% over 15min (page); queue depth >100 (ticket); disk usage on N8N volume >85% (ticket). Every alert ships with a `runbook_url` annotation per the existing CI lint gate (`scripts/check_runbook_url_coverage.py`).
- [ ] **AC7** — Upgrade runbook at `eusolicit-docs/runbooks/n8n-upgrade.md` covering: (a) pre-upgrade workflow export to repo, (b) container stop + image swap + volume mount unchanged, (c) container start with one-minute health-check window, (d) smoke test = trigger a known-good workflow, (e) rollback procedure if smoke-test fails (image tag pinning makes rollback a single `docker compose up` with the previous tag).
- [ ] **AC8** — Practice rebuild executed on a sacrificial Debian 13 VM (NOT production www1 per AP22-D1) using the AC4 + AC7 runbooks; drill results recorded in §Drill Results section of this story file.
- [ ] **AC9** — `bmad-code-review` Approve verdict on the runbook + dashboard + alert-rules PR (two-gate close per AP17-C1) + operator-completion signal after AC8 drill executes successfully.

## Dev / Operator Notes

**Pattern reuse:** This story mirrors the shape of `onprem-01` (Postgres backup/restore), `onprem-02` (Redis persistence/restore), and `onprem-03` (Prometheus/Grafana/Alertmanager) — leverage the same backup destination (Hetzner Storage Box), the same alertmanager.yaml (extend with N8N receivers), the same `scripts/check_runbook_url_coverage.py` CI gate, and the same drill-on-sacrificial-VM AP22-D1 discipline.

**Why this isn't part of E05 amendment:** E05 stories (S05.20–S05.25) cover the *workflow templates* — the JSON specifications that run inside N8N. Operational ownership of the N8N *container* (version, backup, upgrade, observability) is launch-blocking infrastructure work, which is E22's scope. Splitting the concern keeps E05 focused on data-pipeline correctness and E22 focused on operability of the launch substrate.

**Pre-flight:** Verify the current production N8N version (likely already running on www1 via the existing docker-compose stack — confirm with `docker compose ps` on www1 first). The AC1 version-pinning step may be a no-op if the deploy is already pinned; if `latest` is in use, this story closes the implicit-version-drift risk.

**Out of scope:**
- Per-tenant N8N provisioning (out per ADR-018 — N8N is shared at SirmaAI Org level).
- Multi-instance N8N (HA) — single container on www1 per ADR-010 launch posture.
- N8N user/credential admin (managed via existing operator login).
- Workflow-authoring CI (covered by E05 amendment S05.20 PR-review discipline).

## Risk

- **R1**: N8N has historically had breaking changes in minor versions. Pinning + monthly review is the mitigation; major-version bumps must go through sprint-change-proposal review.
- **R2**: Workflow re-import after volume loss requires the credentials store to be re-bindable. Validate AC4 (e) end-to-end in the AC8 drill — this is the most failure-prone restore path.
- **R3**: Prometheus `/metrics` endpoint may need explicit env-var enablement on the N8N container (`N8N_METRICS=true`). Verify before claiming AC5.

## See also

- `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` §ADR-018
- `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md` (ADR-010)
- `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-15.md` Issue #11
- Sibling stories: `onprem-01-postgres-backup-and-recovery.md`, `onprem-02-redis-persistence-and-recovery.md`, `onprem-03-monitoring-on-www1.md`

## Drill Results

*(populated after AC8 execution on sacrificial Debian 13 VM)*

| Step | Expected | Observed | Pass/Fail |
|---|---|---|---|
| Container bootstrap from compose | container Healthy < 60s | — | — |
| Volume restore from Hetzner Storage Box | data dir restored < 5min | — | — |
| Workflow re-import from repo export | all live workflows present | — | — |
| Credential re-bind (SirmaAI token + webhook secret) | smoke workflow runs end-to-end | — | — |
| Upgrade drill (current tag → next minor) | smoke test passes within 1min | — | — |
| Rollback drill (down-tag) | smoke test passes within 1min | — | — |
