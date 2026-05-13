---
title: 'On-Prem Launch Sprint Plan'
date: 2026-05-11
status: Active
author: 'John (PM)'
related:
  - 'eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md'
  - 'eusolicit-docs/planning-artifacts/architecture.md §ADR-010'
  - 'eusolicit-docs/implementation-artifacts/sprint-status.yaml'
---

# On-Prem Launch Sprint Plan — 6 `onprem-*` stories

**Launch target:** 2026-06-01 (per ADR-010 rewrite calendar)
**Sprint scope:** the 6 launch-blocker stories spawned by the 2026-05-11 AWS→single-host pivot.
**Owner:** platform-engineering.
**Not in scope here:** `drift-recovery-story`, `inj-03`, `pe-04`, `pe-06`, `public-sla-announcement-soak-gate` — these stay tracked in `sprint-status.yaml` and have their own owners. Cross-deps with this sprint are called out below.

## Critical finding (audit, 2026-05-11)

**The engineering artifacts for all 6 onprem stories are ~95% written on disk** (scripts, playbooks, prometheus rules, alertmanager config, runbooks, deploy.yml workflow_run trigger, auto-rollback step). They landed in tandem with the ADR rewrite during the same pivot pass.

**What's actually left is operational execution**: drills, host-side cron registration, GitHub UI configuration, credential provisioning on www1. Statuses in `sprint-status.yaml` correctly remain `ready-for-dev` because the AC require these executions, but the work shape is different from a green-field sprint — closer to "verify and rehearse what's already coded."

This document captures the execution sequence + the per-story "what's left."

## Recommended execution order

```
Sprint week 1 (2026-05-13 → 2026-05-15)
  Step 1: onprem-05  Deploy guardrails  (close auto-sync class first;
                                          unblocks safe execution of everything below)
  Step 2: onprem-03  Monitoring stack    (unblocks alert routing for 01/02/04)

Sprint week 2 (2026-05-15 → 2026-05-22)
  Step 3a: onprem-01 Postgres backup+drill   ┐ in parallel
  Step 3b: onprem-02 Redis persist+drill     ┘
  Step 4:  onprem-04 Disk + resource alerts (consumes 03's Prometheus + 21-6's runbooks)

Sprint week 3 (2026-05-18 → 2026-05-22)
  Step 5:  onprem-06 www1-as-code + practice rebuild (consumes 01+02 backups)

Sprint week 3-4 (2026-05-22 → 2026-05-25)
  Step 6:  First post-sprint chaos drill (rescoped pe-04 — outside this sprint)
```

### Why this order

- **05 first** because every subsequent step deploys configs to www1; an auto-sync mid-sprint without the gate would torpedo the work.
- **03 second** because 01/02/04 all need somewhere to fire alerts, and 03 also clears the AWS `cloudwatch-exporter` leftover that contradicts the pivot.
- **01 + 02 parallel** — independent data paths; both produce drill results that 06 consumes.
- **04 after 03** because alerts without Prometheus are dead config.
- **06 last** because the practice rebuild restores from 01+02 backups; rebuilding before backups are proven is theatre.

## Dependency graph

```
                    onprem-05 (deploy guardrails)
                          ↓
                ┌───── onprem-03 (monitoring) ─────┐
                ↓               ↓                   ↓
       onprem-01 ack-wire   onprem-02 visibility  onprem-04 alerts
            ↓                       ↓                   ↓
            └────── first drills ───┴─── first drill ───┘
                                                         ↓
                                          onprem-06 practice rebuild
                                       (consumes 01+02 backup paths)
```

**External cross-deps (outside this sprint):**
- `pe-06-pagerduty-rotation-provisioning` ↔ onprem-03 (shares alertmanager receivers; email + Telegram)
- `pe-04-chaos-drill-execution` depends on onprem-03 + onprem-04 (needs monitoring + alerts to observe drill effects)
- `public-sla-announcement-soak-gate` depends on launch readiness across all 6

## Per-story: artifact state, pending AC, owner action

### Step 1 — onprem-05 (Deploy Guardrails)

| Item | State |
|---|---|
| `deploy.yml` `workflow_run` gate | ✅ on disk (deploy.yml lines 16-26) |
| Auto-rollback step | ✅ on disk (deploy.yml line 178+) |
| Pre-deploy SHA capture for rollback | ✅ on disk (deploy.yml line 77+) |
| `runbooks/branch-protection-policy.md` | ✅ on disk (101 lines) |
| **AC 1**: branch protection on main configured in GitHub UI | ❌ **operator action** |
| **AC 3**: `chore: auto-sync` PR policy | ⚠️ **wrong location** — auto-sync source is the Orchestrator, not a `.github/workflows/` file. AC needs rewriting; work belongs in `Orchestrator/` |
| **AC 5**: dated headers in deploy.log | needs verification in `scripts/deploy.sh` |
| **AC 7**: 30-day no-incident measurement | long-tail; cannot close at sprint end |

**Owner actions to close:**
1. Configure branch protection in GitHub UI (Deb or repo admin).
2. **Decide where AC 3 lives** — either rewrite onprem-05 AC 3 to point at the Orchestrator, OR open a paired `orchestrator-side` story. See "Open question A" below.
3. Verify `scripts/deploy.sh` log format change landed; if not, one-line patch.

---

### Step 2 — onprem-03 (Monitoring on www1)

| Item | State |
|---|---|
| `infra/observability/prometheus/prometheus.yml` | ✅ on disk |
| `infra/observability/prometheus/rules/` (4 files) | ✅ on disk including host-alerts.yaml |
| `infra/observability/alertmanager/alertmanager.yaml` | ✅ on disk — pivot-aware (email+Telegram+Slack, no PagerDuty receivers visible) |
| 8 Grafana dashboard JSONs | ✅ on disk |
| `infra/observability/docker-compose.observability.yml` | ✅ on disk |
| `runbooks/observability-stack.md` | ✅ on disk (85 lines) |
| **AC 1**: reuse `lifematch-dev-*` vs separate stack — decided + executed | ❌ choice not yet made |
| **AC 7**: nginx reverse proxy for Grafana | ❌ operator action on www1 |
| `infra/observability/cloudwatch-exporter/` | ⚠️ **dead code from pre-pivot — should be deleted** |

**Owner actions to close:**
1. Make the reuse-vs-separate decision; execute it.
2. nginx site config for `grafana.eusolicit.internal` + auth.
3. Delete `infra/observability/cloudwatch-exporter/`.
4. Verify all 7 service `up=1` in Prometheus.

---

### Step 3a — onprem-01 (Postgres Backup & Recovery)

| Item | State |
|---|---|
| `scripts/onprem/postgres-backup.sh` | ✅ on disk (restic-based, daily + WAL-only) |
| `scripts/onprem/postgres-restore.sh` | ✅ on disk |
| `scripts/onprem/postgres-restore-test.sh` | ✅ on disk |
| `runbooks/postgres-restore.md` | ✅ on disk (239 lines) |
| `infra/host/cron.d/` | ✅ directory exists with cron skeletons |
| **AC 2**: Hetzner Storage Box credentials in `.env.backup` on www1 | ❌ host-only — operator provisions |
| **AC 4**: cron registered on www1 at `/etc/cron.d/eusolicit-backup` | ❌ host-only — operator action |
| **AC 5**: weekly restore-test alert routing | depends on **onprem-03** |
| **AC 8**: first full restore drill executed | ❌ `runbooks/postgres-restore.md §First Drill Results` = `TBD` |

**Owner actions to close:**
1. Provision Hetzner Storage Box subaccount; populate `.env.backup` on www1 (host-only file per memory note).
2. Install cron entry on www1.
3. Run first restore drill; record measured RTO in §First Drill Results.

---

### Step 3b — onprem-02 (Redis Persistence & Recovery)

| Item | State |
|---|---|
| `scripts/onprem/redis-backup.sh` | ✅ on disk |
| `scripts/onprem/redis-restore.sh` | ✅ on disk |
| `runbooks/redis-restore.md` | ✅ on disk (135 lines) |
| **AC 1**: `redis.conf` mounted in docker-compose.prod.yml (AOF + RDB) | ⚠️ AC asserts mount; verify against `docker-compose.prod.yml` |
| **AC 4**: 6-service reconnect post-restart | needs onprem-03 monitoring for observability |
| **AC 6**: first controlled-restart drill executed | ❌ `runbooks/redis-restore.md §First Drill Results` = `TBD` |

**Owner actions to close:**
1. Confirm `redis.conf` mount path in `docker-compose.prod.yml` (one-grep verification).
2. Schedule low-traffic window for restart drill; execute; record results.

---

### Step 4 — onprem-04 (Disk & Resource Monitoring)

| Item | State |
|---|---|
| `infra/observability/prometheus/rules/host-alerts.yaml` | ✅ on disk (129 lines, 10 alerts with `runbook_url`) |
| 7 runbooks (disk-cleanup-root/home, memory-pressure, container-restart-loop, wal-archiving-stalled, docker-daemon-recovery, ntp-drift) | ✅ all on disk |
| **AC 1/2**: `node_exporter` + `cAdvisor` containers in observability compose | ❌ needs adding to compose |
| **AC 11**: `scripts/check_runbook_url_coverage.py` gate green in CI | needs CI run to verify |
| **AC 12**: each alert fired in contrived scenario | ❌ operator action |

**Owner actions to close:**
1. Add `node_exporter` + `cAdvisor` services to `docker-compose.observability.yml`.
2. CI verify `check_runbook_url_coverage.py` passes for new rules.
3. Fire each alert via contrived scenario (e.g., balloon `/tmp` for disk; chaos-kill a container for restart-loop).

---

### Step 5 — onprem-06 (www1 as Code)

| Item | State |
|---|---|
| `infra/host/` (ansible.cfg, inventory.yaml, README.md, templates/, eusolicit-overrides/, cron.d/) | ✅ on disk |
| 6 playbooks + `site.yml` | ✅ on disk (avg 60-80 lines each; not skeletons) |
| `runbooks/www1-rebuild.md` | ✅ on disk (181 lines) |
| **AC 5**: docker daemon storage-root move to `/home/docker` | ❌ **dangerous, needs maintenance window on www1** — separate sub-task |
| **AC 8**: practice rebuild on sacrificial Debian 13 VM | ❌ `runbooks/www1-rebuild.md §First Rebuild Drill Results` = `TBD` |
| `authorized_keys` real keys + `.env.prod` etc. | ❌ host-only by design — `.example` templates only in repo |

**Owner actions to close:**
1. Schedule maintenance window for storage-root migration (consider AT THE SAME TIME as restore drill — you're touching the host anyway).
2. Provision Hetzner Cloud trial VM or local KVM; execute `playbooks/site.yml` + restore drill against it; record measured RTO.

---

## Open questions — resolved 2026-05-11

### A. onprem-05 AC 3 — auto-sync workflow location — ✅ RESOLVED

**Resolution:** Option A1 — onprem-05 AC 3 rewritten to point at the Orchestrator. The auto-sync source is `Orchestrator/`, not `eusolicit-app/.github/workflows/`. AC 3 closes when the first `chore: auto-sync` PR opens against `main` and merges through human review. Tracking now lives partly in `Orchestrator/` task list; the eusolicit-app branch protection (AC 1) covers the "no self-approve" enforcement on the merge side.

Files edited: `onprem-05-deploy-guardrails.md` (AC 3 + Task 3).

### B. `infra/helm/` + `infra/observability/cloudwatch-exporter/` disposition — ✅ RESOLVED

**Resolution:** both AWS-era leftovers folded into onprem-03 scope as Tasks 9 and 10.

- **cloudwatch-exporter** — delete; rename/refactor `test_pe05_cloudwatch_and_postgres_exporter.py` to preserve postgres_exporter coverage (still in scope per onprem-03 AC 5).
- **`infra/helm/`** — move to `infra/helm/.archive/` rather than delete (preserves design for hypothetical Phase-2 HA migration). Affects 4 test files + `check_helm_pdb_and_minreplicas.py` lint gate + 2 READMEs.

Files edited: `onprem-03-monitoring-on-www1.md` (Tasks 9 + 10 added).

### C. `pe-02-cutover-runbook.md` / `pe-03-cutover-runbook.md` archival — ✅ ALREADY DONE

Verified on disk:
- `eusolicit-docs/implementation-artifacts/.archive/pe-02-cutover-runbook.md` exists with `⚠ SUPERSEDED 2026-05-11` header on line 3
- `eusolicit-docs/implementation-artifacts/.archive/pe-03-cutover-runbook.md` exists with same marker
- Top-level versions removed

No action required.

## Sprint-status.yaml reconcile (read-only audit)

Per audit 2026-05-11, the 6 entries' `ready-for-dev` status remains correct — operational AC remain open in every case. **No surgical edits recommended** for the 6 onprem entries.

Sprint-status.yaml `last_updated` is 2026-05-04 — a week stale. Items potentially completed since then (notably the artifact landings for all 6 onprem stories) are NOT yet reflected in this file. If the orchestrator re-runs and regenerates statuses, expect noise — per memory note, do not let it regenerate; surgical edits only.

## Calendar (carry-forward from ADR-010 calendar, normalized)

| Date range | Milestone |
|---|---|
| 2026-05-13 → 2026-05-15 | Step 1 (onprem-05) + Step 2 (onprem-03) |
| 2026-05-15 → 2026-05-18 | Steps 3a/3b/4 (onprem-01, 02, 04) |
| 2026-05-18 → 2026-05-22 | Step 5 (onprem-06 + practice rebuild) |
| 2026-05-22 → 2026-05-25 | First chaos drill (pe-04, outside this sprint) |
| 2026-05-25 → 2026-05-29 | inj-03 + drift-recovery AC5 (outside this sprint) |
| **2026-06-01** | **Launch target** |

## What I'm tracking, John

- AC drift: any new operator-found gap during execution should land in this doc, not in a fresh report.
- Drill results: when each `§First Drill Results` section is filled, that's the closure signal for the related story.
- onprem-05 AC 3 resolution: pending Deb's call on A1 vs A2.
