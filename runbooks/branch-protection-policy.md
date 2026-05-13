# Branch Protection Policy — `main` (Story onprem-05)

**Status:** Partially landed 2026-05-11 — see "Currently enabled" below
**Source story:** `eusolicit-docs/implementation-artifacts/onprem-05-deploy-guardrails.md`
**Related:** `.github/workflows/deploy.yml` (now gates on CI via `workflow_run` trigger)

## Currently enabled (2026-05-11)

After the GitHub subscription upgrade, the following minimum rule landed on `main`:

- ✅ **Require status checks to pass** — single top-level `CI` check.

Deferred (operator decision):

- ❌ Require PR + approval before merge — would block auto-sync direct-pushes; not enabled until auto-sync redirect lands (AC3).
- ❌ Restrict force pushes — **recommended next toggle**, single click, no subscription tier issue.
- ❌ Restrict deletions — **recommended next toggle**.
- ❌ Block admin bypass.
- ❌ Require linear history.
- ❌ Require signed commits.

**Net protection today:** PR-merge path is CI-gated. Direct push to `main` is still possible (auto-sync still works). Force-push to `main` is still possible — this is the highest residual irreversible-damage risk and is worth closing with a single click. Production deploy is independently protected by `deploy.yml`'s `workflow_run` gate: a red CI run blocks the deploy regardless of how the commit landed on `main`.

## Why this exists

The 2026-05-09 / 2026-05-10 4-hour SEV1 was caused by `chore: auto-sync 2026-05-09 18:31:59` (commit `ac2fd08`) shipping lint- and build-broken code straight to production. The deploy pipeline at the time fired on every push to `main` with no CI dependency — a broken commit was deployed before CI even finished.

The companion repo change in commit `fb42f3e` modifies `.github/workflows/deploy.yml` so that the deploy job:
1. Triggers via `workflow_run` after CI completes (not on raw `push`).
2. Gates on `github.event.workflow_run.conclusion == 'success'`.
3. Captures the pre-deploy SHA on `www1` and auto-rolls-back if smoke tests fail.

The workflow change alone is sufficient to gate deployment on CI for the **workflow_run** path. But branch protection is the belt-and-suspenders that prevents a contributor from bypassing CI via a force-push, an admin override, or a direct push that races CI completion. **Both** are required for a full close-out of AC1 + AC2.

## What to configure (operator action)

GitHub repo: `<org>/eusolicit-app` → **Settings → Branches → Branch protection rules**

Add a new rule (or edit existing) for branch pattern `main`:

1. **Require a pull request before merging** ✅
   - Require approvals: **1** (minimum)
   - Dismiss stale pull request approvals when new commits are pushed: ✅
   - Require review from Code Owners: optional (enable if CODEOWNERS file is in use)

2. **Require status checks to pass before merging** ✅
   - Require branches to be up to date before merging: ✅
   - Required status checks (search and add each):
     - `CI / check (client-api)`
     - `CI / check (admin-api)`
     - `CI / check (data-pipeline)`
     - `CI / check (ai-gateway)`
     - `CI / check (notification)`
     - `CI / check (integrations-api)`
     - `CI / check (eusolicit-common)`
     - `CI / check (eusolicit-models)`
     - `CI / check (eusolicit-test-utils)`
     - `CI / validate-trust-artefacts-yaml`
     - `CI / helm-pdb-lint` ⚠ NOTE: this gate became dead-code after the on-prem pivot (helm chart deleted). Either remove the gate from `ci.yml` or keep it as a soft-check; do not require it here until that decision is made.
     - `CI / runbook-url-coverage`
     - `CI / validate-sub-processors-yaml`
     - `CI / check-subprocessor-changelog`
     - `CI / enforce-subprocessor-advance-notice`

3. **Require conversation resolution before merging** ✅

4. **Require signed commits** — recommended but not required.

5. **Require linear history** ✅ — keeps `main` clean for git archaeology.

6. **Do not allow bypassing the above settings** ✅ (includes administrators).

7. **Restrict who can push to matching branches** — leave unchecked (PR-only is the gate).

## Auto-sync commit policy

`chore: auto-sync …` commits historically pushed directly to `main`. Under this policy that becomes impossible (PR + 1 approval + CI green required). The upstream automation that generates auto-sync commits must be reconfigured to **open a PR** instead. Until that upstream change lands, auto-sync commits will fail to merge — which is the safer failure mode and exactly what we want.

If the upstream auto-sync system can be configured but a human gate is undesirable, an alternative is to grant a dedicated bot account the bypass right on this branch protection rule. **Do not do this.** The whole point of the rule is to close the auto-sync outage class; granting the bot a bypass reinstates the original failure mode.

## Verification

After enabling the rule:

1. **Push a deliberately failing branch** (e.g., a lint error) to a feature branch, open a PR. Confirm:
   - Merge button is disabled while CI is running.
   - Merge button stays disabled when CI fails.
   - Merge button enables once CI passes (after a fix).

2. **Try a direct push to `main`** from a non-admin account. Should be rejected with "Cannot push to protected branch."

3. **Trigger `workflow_dispatch` on `deploy.yml`** from the Actions tab. Should still work — this is the operator-override path (e.g., for re-running a known-good deploy after manual intervention).

4. **Confirm `deploy.yml` does not fire on a PR merge if CI fails.** Push a PR that lands a failing CI; confirm CI fails; squash-merge regardless (the PR may still merge if you bypass as admin); confirm `deploy.yml` does NOT run (the `if:` gate on `workflow_run.conclusion == 'success'` blocks it).

## References

- Story: `eusolicit-docs/implementation-artifacts/onprem-05-deploy-guardrails.md`
- Workflow: `eusolicit-app/.github/workflows/deploy.yml`
- Incident log: `eusolicit-docs/incident-management/auto-sync-incidents.md`
- Project memory: "Auto-sync ships unverified code"
