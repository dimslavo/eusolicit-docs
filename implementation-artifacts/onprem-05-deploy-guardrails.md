# Story onprem-05: Deploy Guardrails

Status: ready-for-dev

**Origin:** Onprem pivot decision (eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md). **Closes the auto-sync outage class** — the actual cause of the 2026-05-09/10 four-hour SEV1.
**Priority:** P0 — launch-blocker. Without this, the next `chore: auto-sync` commit breaks prod again.
**Epic:** Post-Epic-21 / Onprem Launch
**Points:** 1 | **Type:** infra / ci-cd

## Story

As a **platform-engineering operator**,
I want **the GitHub Actions `deploy.yml` workflow to be gated on green CI, `chore: auto-sync` PRs to require human review, smoke-test failures to roll back automatically, and `deploy.log` on www1 to include dated headers**,
so that **the recurring outage pattern of "auto-sync ships unverified code → prod breaks → 4h+ recovery" is structurally eliminated, not just documented in retrospective notes**.

## Acceptance Criteria

1. **Branch protection on `main` requires CI green** — GitHub repo settings → Branches → main → Require status checks to pass before merging → "CI / check" required. Configured manually via the GitHub UI (operator action) and documented in `eusolicit-docs/runbooks/branch-protection-policy.md` (NEW).

2. **`deploy.yml` only fires after CI passes** — currently `deploy.yml` and `ci.yml` run in parallel on push to main; the deploy job has no `needs:` dependency on CI. Change: add `concurrency` + `if: github.event.workflow_run.conclusion == 'success'` pattern, OR convert deploy.yml to trigger on `workflow_run: workflows: [CI]` instead of `push: branches: [main]`. Test by intentionally pushing a failing change and confirming deploy does NOT run.

3. **`chore: auto-sync` PR policy** — auto-sync commits must land via PR (not direct push). The auto-sync workflow that currently pushes directly is modified to open a PR instead. The PR template states "Requires human review before merge — auto-sync is unverified code per project memory." The PR author cannot self-approve (GitHub setting: require review from someone other than the author).

4. **Auto-rollback on smoke-test failure** — `deploy.yml`'s smoke-test step (already present, currently only logs) is extended to:
   - On failure: `git reset --hard <previous-sha>` on www1, re-run `bash scripts/deploy.sh` to redeploy the previous version, exit non-zero with a clear "ROLLED BACK to <sha>" message.
   - The `<previous-sha>` is captured BEFORE the new deploy starts (e.g., `git rev-parse HEAD` and saved to a tempfile or env var).
   - Alertmanager (deps `onprem-03`) receives the rollback event.

5. **`deploy.log` includes dated headers** — current log lines start with `[HH:MM:SS]` only (per direct file inspection on www1). Update `scripts/deploy.sh` to emit `[YYYY-MM-DD HH:MM:SS]` so log archeology across days works. Add a section divider line `========== Deploy <SHA> at <DATE> ==========` at the start of each deploy invocation.

6. **Deploy approval annotation** — every `deploy.yml` run posts a GitHub deployment status to the relevant commit, including the deploy SHA, smoke-test results, and rollback status if applicable. This gives `git log` linkable evidence of "what was actually deployed when."

7. **Auto-sync incident recurrence: zero in 30 days post-merge** — measurable acceptance criterion. Track via a `eusolicit-docs/incident-management/auto-sync-incidents.md` log; if any auto-sync commit causes a prod outage between merge of this story and 30 days later, the story fails its long-term AC and a follow-up ticket opens.

## Tasks / Subtasks

- [ ] Task 1: Configure branch protection on main via GitHub UI (operator action; not codifiable in this repo, but document the setting in `branch-protection-policy.md`).
- [ ] Task 2: Modify `.github/workflows/deploy.yml` to depend on CI conclusion (workflow_run trigger or status-check gate).
- [ ] Task 3: Modify the auto-sync workflow (`.github/workflows/<auto-sync-name>.yml` — find it) to open PRs instead of pushing directly.
- [ ] Task 4: Add a rollback step to `deploy.yml` smoke-test on-failure path.
- [ ] Task 5: Update `scripts/deploy.sh` log format (dated headers + section dividers).
- [ ] Task 6: Test with an intentional failing push to a branch + verify deploy.yml does NOT run.
- [ ] Task 7: Test rollback with an intentional bad commit that passes CI but fails smoke-tests.
- [ ] Task 8: Document the policy in `branch-protection-policy.md` + update `eusolicit-docs/incident-management/auto-sync-incidents.md`.

## Dev Notes

### Why this is a P0

Project memory note: "Auto-sync ships unverified code — `chore: auto-sync …` commits routinely break builds; expect multiple defects per commit." The most recent example, `ac2fd08 chore: auto-sync 2026-05-09 18:31:59`, caused a 4-hour SEV1 on 2026-05-10. The follow-up commits `e52bb40 fix(frontend): unblock production build — type/lint errors from auto-sync` and `27487a7 fix(deploy): unbreak prod /api/v1, retire admin host TLS error, clear ac2fd08 lint/type debt` are the recovery trail. This is not theoretical — it's the recurring failure mode of the deploy pipeline.

### Current deploy.yml flow (per `.github/workflows/deploy.yml`)

```yaml
on:
  push:
    branches: [main]
```

Deploys on every push, no CI dependency. The "NOTE" comment at line 35-39 in the workflow file says:
> "This workflow runs in parallel. The deploy job uses a GitHub Environment ('production') which can optionally require CI to pass first via branch protection rules (Settings → Branches → main → Require status checks: 'CI / check')."

So the workaround exists but is configured manually on GitHub, not in code. AC 1 + AC 2 close this gap by both configuring the GitHub setting AND making the workflow itself dependent.

### Smoke test currently only logs

`deploy.yml` lines 78–153 implement smoke tests but on `if: failure()` (line 158), the "Notify on failure" step only runs `docker compose ps` + `logs --tail=30` — no rollback. AC 4 fixes this.

### References

- ADR-010 rewrite: `architecture.md` §ADR-010 (2026-05-11)
- Decision record: `onprem-pivot-decision-2026-05-11.md`
- Project memory: "Auto-sync ships unverified code"
- Failed deploys evidence: `eusolicit-app/deploy.log` lines around `[18:53:02] ERROR: Migration failed for data-pipeline`
- `.github/workflows/deploy.yml`
- `.github/workflows/ci.yml`

### Out of scope

- Replacing the deploy pipeline with a different platform (e.g., ArgoCD, GitOps) — that's a Phase-2 question.
- Canary / blue-green deploys — single-host posture doesn't naturally support these.
- Hardening the auto-sync source itself (the upstream automation that generates `chore: auto-sync` commits). That's an orchestrator-level concern, outside this story.
