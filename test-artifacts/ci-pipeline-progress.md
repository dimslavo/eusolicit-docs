---
stepsCompleted:
  - 'step-01-preflight'
  - 'step-02-generate-pipeline'
  - 'step-03-configure-quality-gates'
  - 'step-04-validate-and-summary'
lastStep: 'step-04-validate-and-summary'
lastSaved: '2026-04-19'
ciPlatform: 'github-actions'
testStackType: 'fullstack'
testFrameworks:
  - 'playwright'
  - 'pytest'
pythonVersion: '3.12'
nodeVersion: '22'
pnpmVersion: '10.33.0'
---

# CI/CD Pipeline Progress — EU Solicit

## Step 1: Preflight

**Date:** 2026-04-19
**Result:** ✅ All preflight checks passed

| Check | Status | Detail |
|-------|--------|--------|
| Git repository | ✅ | `.git/` exists in `eusolicit-app/` |
| CI platform | ✅ | `github-actions` — `.github/workflows/` detected (5 workflow files) |
| Test stack type | ✅ | `fullstack` — both `playwright.config.ts` and `pyproject.toml[tool.pytest]` present |
| Test framework (E2E) | ✅ | Playwright — `playwright.config.ts` with `e2e/` directory |
| Test framework (Backend) | ✅ | pytest — `pyproject.toml` with `[tool.pytest.ini_options]` |
| Python version | ✅ | 3.12 — `.python-version` file + `pyproject.toml` |
| Node version | ✅ | v22 — `.nvmrc` |
| Package manager | ✅ | pnpm@10.33.0 — `frontend/package.json` `packageManager` field |

---

## Step 2: Pipeline Generation / Audit

**Date:** 2026-04-19

The CI/CD pipeline was previously scaffolded and is comprehensive. This run performed an audit and applied one bug fix.

### Existing Workflow Files

| File | Purpose | Status |
|------|---------|--------|
| `.github/workflows/ci.yml` | 8-job parallel matrix: per-project lint + type-check + unit tests; Docker API tests on `push main` | ✅ Reviewed |
| `.github/workflows/test.yml` | Full test pipeline: quality-gate → smoke → backend → E2E (4 shards) → burn-in → report | ✅ Fixed |
| `.github/workflows/quality-gates.yml` | PR blocking gates: code-quality (P0), coverage ≥80% (P0), burn-in-stability (P1) | ✅ Fixed |
| `.github/workflows/nightly.yml` | Nightly: agent error tests (matrix) + k6 load smoke test | ✅ Reviewed |
| `.github/workflows/deploy.yml` | SSH deploy to `www1.endigitalx.com` with post-deploy health checks | ✅ Reviewed |

### Bug Fixed: pnpm Version Mismatch

**Issue:** `test.yml` and `quality-gates.yml` both specified `pnpm/action-setup@v4` with `version: 9`, but the project uses `pnpm@10.33.0` (declared in `frontend/package.json` `packageManager` field). Installing pnpm v9 against a v10 lockfile would cause `pnpm install --frozen-lockfile` to fail.

**Fix:** Updated `version: 9` → `version: "10"` in both workflow files.

Files changed:
- `.github/workflows/test.yml` — pnpm version 9 → "10"
- `.github/workflows/quality-gates.yml` — pnpm version 9 → "10"

### Pipeline Architecture (Sequential Flow)

```
CI (ci.yml)         — push/PR (parallel, always)
    8×  lint + type-check + unit (per project)
    1×  API tests via Docker Compose (push main only)

Test Pipeline (test.yml)  — push/PR/schedule
    quality-gate (lint + types: Python + Frontend)
        └── smoke-test (Docker Compose + E2E smoke)
                ├── backend-test (pytest unit + integration, Postgres+Redis services)
                └── e2e-test ×4 shards (Playwright Chromium, Docker Compose)
                            └── burn-in (schedule Sun 02:00 UTC or run-burn-in label)
                                        └── report (always)

Quality Gates (quality-gates.yml)  — all PRs to main/develop
    code-quality (P0: ruff + mypy + tsc, zero tolerance)
        ├── coverage-gate (P0: pytest --cov ≥80%, Postgres+Redis services)
        └── burn-in-gate (P1: scripts/burn-in-changed.sh 5 iterations)
    quality-summary (always, reports pass/fail table)

Nightly (nightly.yml)  — Mon–Fri 03:00 UTC
    agent-error-tests ×2 (client-api, admin-api pytest, no Docker)
    k6-load-smoke (Docker Compose stack + k6 10 VUs 60s)

Deploy (deploy.yml)  — push main / workflow_dispatch
    SSH deploy → git pull → deploy.sh → health checks
```

---

## Step 3: Quality Gates Configuration

**Date:** 2026-04-19

### Burn-In Strategy

- **Stack type**: `fullstack` — burn-in enabled (targets UI flakiness)
- **Test Pipeline burn-in**: Runs full Playwright suite ×5 on weekly schedule or `run-burn-in` PR label
- **Quality Gates burn-in**: Runs `scripts/burn-in-changed.sh 5 "origin/$BASE_REF"` on any PR with changed test specs
- **Script**: `scripts/burn-in-changed.sh` — detects changed E2E and Python test files, runs them N times

Security note: `BASE_REF` from `github.base_ref` is passed via `env:` intermediary (not interpolated directly) — complies with script injection prevention rules.

### Quality Thresholds

| Gate | Priority | Threshold | Action on Failure |
|------|----------|-----------|-------------------|
| Ruff lint | P0 | Zero violations | Merge blocked |
| Mypy type check | P0 | Zero errors | Merge blocked |
| Frontend ESLint | P0 | Zero violations | Merge blocked |
| TypeScript compile | P0 | Zero errors | Merge blocked |
| pytest coverage | P0 | ≥ 80% (from `pyproject.toml`) | Merge blocked |
| Burn-in stability | P1 | 5/5 iterations pass | Merge blocked |

### Notifications

Slack notification placeholder exists in `quality-gates.yml` (commented). Enable by:
1. Creating `SLACK_WEBHOOK_URL` secret
2. Uncommenting the `slackapi/slack-github-action` step in `quality-summary` job

---

## Step 4: Validation & Summary

**Date:** 2026-04-19

### Checklist Status

| Category | Item | Status |
|----------|------|--------|
| **Preflight** | Git repository | ✅ |
| **Preflight** | CI platform detected | ✅ github-actions |
| **Preflight** | Test framework configured | ✅ pytest + playwright |
| **Preflight** | Node version identified | ✅ v22 (.nvmrc) |
| **Pipeline** | CI config file exists at correct path | ✅ 5 workflow files |
| **Pipeline** | Correct test commands | ✅ pytest + npx playwright |
| **Pipeline** | Node/Python versions match project | ✅ |
| **Pipeline** | Browser install for fullstack | ✅ chromium in smoke + e2e jobs |
| **Pipeline** | pnpm version correct | ✅ Fixed: v10 |
| **Sharding** | Matrix strategy configured | ✅ 4 shards in e2e-test |
| **Sharding** | fail-fast: false | ✅ |
| **Burn-In** | Burn-in enabled (fullstack) | ✅ |
| **Burn-In** | 5 iterations in quality-gates | ✅ |
| **Burn-In** | Failure artifacts uploaded | ✅ burn-in-gate-failures |
| **Caching** | pip cache with lockfile key | ✅ |
| **Caching** | Playwright browser cache | ✅ |
| **Caching** | Docker layer cache | ✅ ci.yml integration job |
| **Artifacts** | JUnit XML | ✅ Multiple jobs |
| **Artifacts** | Retention days set | ✅ 7–30 days |
| **Artifacts** | Failure-sensitive collection | ✅ Some upload on failure |
| **Scripts** | test-changed.sh | ✅ |
| **Scripts** | ci-local.sh | ✅ |
| **Scripts** | burn-in-changed.sh | ✅ |
| **Scripts** | Executable | ✅ -rwxrwxr-x |
| **Docs** | docs/ci.md | ✅ Updated (all 5 workflows) |
| **Docs** | docs/ci-secrets-checklist.md | ✅ Updated (DEPLOY_SSH_KEY added) |
| **Security** | inputs through env: intermediary | ✅ |
| **Security** | No direct ${{ inputs.* }} in run: | ✅ |

### Completion Summary

**CI Platform:** GitHub Actions  
**Config Paths:** `.github/workflows/` (5 files: `ci.yml`, `test.yml`, `quality-gates.yml`, `nightly.yml`, `deploy.yml`)

**Key stages enabled:**
- ✅ Per-project lint + type-check + unit test matrix (8-job parallel)
- ✅ Full test pipeline with smoke → backend → E2E (4 shards) → burn-in
- ✅ PR blocking quality gates (P0: lint/types/coverage, P1: burn-in)
- ✅ Nightly agent error tests + k6 load smoke
- ✅ SSH deploy with post-deploy health checks

**Artifacts:** JUnit XML, coverage.xml, Playwright HTML report, k6 JSON — 7–30 day retention  
**Notifications:** Slack placeholder available in quality-gates.yml

### Changes Made in This Run

1. **`.github/workflows/test.yml`** — Fixed pnpm version `9` → `"10"`
2. **`.github/workflows/quality-gates.yml`** — Fixed pnpm version `9` → `"10"`
3. **`docs/ci.md`** — Rewrote to document all 5 workflows, added pipeline architecture diagram, artifact table, badge URLs, troubleshooting guide
4. **`docs/ci-secrets-checklist.md`** — Added critical `DEPLOY_SSH_KEY` secret (deploy.yml requires it), added setup instructions, reorganized by category

### Next Steps

1. **Commit and push** the changed files (`test.yml`, `quality-gates.yml`, `docs/ci.md`, `docs/ci-secrets-checklist.md`)
2. **Configure `DEPLOY_SSH_KEY`** secret in GitHub repository Settings — required for `deploy.yml` to work
3. **Create `production` GitHub Environment** with required reviewers and status checks
4. **Trigger CI** by opening a PR or pushing to `develop`
5. **Configure optional `SLACK_WEBHOOK_URL`** secret and uncomment notification step in `quality-gates.yml` when ready
