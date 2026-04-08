---
stepsCompleted:
  - 'step-01-preflight-and-context'
  - 'step-02-generation-mode'
  - 'step-03-test-strategy'
  - 'step-04-generate-tests'
  - 'step-04c-aggregate'
  - 'step-05-validate-and-complete'
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-06'
workflowType: 'testarch-atdd'
storyId: '1-8-github-actions-ci-pipeline'
detectedStack: 'infrastructure'
generationMode: 'ai-generation'
executionMode: 'sequential'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/1-8-github-actions-ci-pipeline.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-01.md'
  - 'eusolicit-app/pyproject.toml'
  - 'eusolicit-app/.github/workflows/test.yml'
  - 'eusolicit-app/.github/workflows/quality-gates.yml'
  - 'eusolicit-app/services/client-api/pyproject.toml'
  - 'eusolicit-app/packages/eusolicit-common/pyproject.toml'
---

# ATDD Checklist: Story 1.8 -- GitHub Actions CI Pipeline

**Date:** 2026-04-06
**Story:** 1-8-github-actions-ci-pipeline
**TDD Phase:** RED (failing tests -- implementation pending)
**Stack:** Infrastructure (GitHub Actions YAML + Python TOML config)

---

## Preflight Summary

| Item | Status |
|------|--------|
| Story approved with clear ACs | ✅ 9 acceptance criteria |
| Test framework configured | ✅ pytest + tomllib (stdlib) + pyyaml |
| Dev environment available | ✅ Monorepo with Python 3.12 |
| Detected stack | `infrastructure` (CI config + YAML + TOML validation) |
| Generation mode | AI generation (static config file validation) |
| Epic test design loaded | ✅ test-design-epic-01.md |
| TEA config flags | None configured -- infrastructure validation tests only |

---

## TDD Red Phase Status

✅ **Failing tests generated — 57 failing, 21 passing**

| Test File | Test Count | Failed | Passed | AC Coverage | Priority |
|-----------|-----------|--------|--------|-------------|----------|
| `tests/unit/test_ci_ruff_config.py` | 13 | 13 | 0 | AC 4 | P0/P1 |
| `tests/unit/test_ci_mypy_config.py` | 7 | 7 | 0 | AC 5 | P0/P1 |
| `tests/unit/test_ci_workflow_structure.py` | 28 | 28 | 0 | AC 1, 2, 3, 6, 7, 8, 9 | P0/P1/P2 |
| `tests/unit/test_ci_dev_dependencies.py` | 30 | 9 | 21 | AC 3, 6 | P1 |
| **Total** | **78** | **57** | **21** | **AC 1-9** | |

**21 passing tests** are service dev dependency checks (services already have complete `[dev]` extras). The 9 failing dev dependency tests are for shared packages that need `ruff` and `mypy` added to their `[dev]` extras, and `eusolicit-common` which has no `[dev]` extras at all.

---

## Acceptance Criteria Coverage Matrix

| AC | Description | Test File | Scenarios | Priority |
|----|-------------|-----------|-----------|----------|
| **AC 1** | Triggers on push to main + all PRs | `test_ci_workflow_structure.py` | push-to-main trigger, pull_request trigger | **P0** |
| **AC 2** | Matrix strategy: 8 jobs in parallel | `test_ci_workflow_structure.py` | matrix strategy defined, 8 entries, each of 8 project names present, fail-fast false | **P0** |
| **AC 3** | Per-job checks: ruff, mypy, pytest | `test_ci_workflow_structure.py`, `test_ci_dev_dependencies.py` | ruff step exists, mypy step exists, pytest step exists, Python 3.12 configured, dev extras include test tools | **P0** |
| **AC 4** | Ruff config with I, E, W, F, UP rules | `test_ci_ruff_config.py` | file exists, valid TOML, 5 required rule sets, target-version py312, line-length, exclude patterns, per-file-ignores S101 | **P0** |
| **AC 5** | Mypy config at repo root with overrides | `test_ci_mypy_config.py` | [tool.mypy] exists, python_version 3.12, warn_return_any, warn_unused_configs, ignore_missing_imports, per-package overrides, test files override | **P0/P1** |
| **AC 6** | Pip caching keyed by pyproject.toml hash | `test_ci_workflow_structure.py` | actions/cache used, cache key references pyproject.toml | **P1** |
| **AC 7** | Docker layer caching for integration tests | `test_ci_workflow_structure.py` | Docker caching configured or integration job present | **P2** |
| **AC 8** | Failure reporting shows which check failed | `test_ci_workflow_structure.py` | Step names include matrix.project (≥3 steps: lint, type-check, test) | **P1** |
| **AC 9** | Workflow completes in under 10 minutes | `test_ci_workflow_structure.py` | timeout-minutes ≤ 10, concurrency group defined, cancel-in-progress true | **P1** |

---

## Risk Coverage

| Risk ID | Description | Score | Test Coverage |
|---------|-------------|-------|---------------|
| **E01-R-006** | CI pipeline matrix build timeout — 8-job matrix with pip install + mypy may exceed 10-minute target | 4 (MEDIUM) | Timeout ≤10min validated; pip caching with pyproject.toml hash verified; concurrency with cancel-in-progress ensures no stale runs; fail-fast false ensures all 8 jobs complete; all 8 matrix projects verified present |

---

## Priority Distribution

| Priority | Count | Description |
|----------|-------|-------------|
| **P0** | 41 | Workflow existence, triggers, matrix 8 entries, per-job checks (ruff/mypy/pytest), ruff config (5 rule sets), mypy config existence |
| **P1** | 33 | Ruff target-version/line-length/exclusions/per-file-ignores, mypy settings/overrides, pip caching, failure reporting, performance settings, actions versions, dev dependencies |
| **P2** | 4 | Docker layer caching |
| **Total** | **78** | |

---

## Test Strategy Notes

### Why Static Config Validation Tests

Story 1-8 creates **configuration files** (ruff.toml, ci.yml, mypy config) and modifies pyproject.toml — not application code. All tests are static file validation:

- `@pytest.mark.unit` marker on all tests
- No Docker Compose required
- No integration tests (no CI runner needed)
- Pure file-system reads + TOML/YAML parsing + assertion
- Uses `tomllib` (Python 3.11+ stdlib) for TOML parsing
- Uses `pyyaml` for YAML parsing of ci.yml

### Why No E2E/Integration Tests

- The CI workflow itself is the integration test — it runs on GitHub Actions
- Validating the workflow YAML structure locally catches configuration errors before push
- Actual CI execution is verified by the P0 epic-level test: "push to main → matrix build completes"
- Runtime behavior (cache hits, Docker layer caching speed) can only be validated in CI

### Failure Mechanisms

Tests fail via three mechanisms:

1. **File missing**: `ruff.toml` and `ci.yml` don't exist → config loads as `None` → every dependent test fails with assertion "not found"
2. **Section missing**: `pyproject.toml` exists but `[tool.mypy]` absent → config is `None` → all mypy tests fail
3. **Incomplete config**: Package `pyproject.toml` files exist but lack complete `[dev]` extras → specific dependency tests fail

When the developer starts implementing:

1. **Task 1 (ruff.toml)**: Create the file → all 13 ruff config tests should pass
2. **Task 2 (mypy config)**: Add `[tool.mypy]` to pyproject.toml → all 7 mypy tests should pass
3. **Task 3 (ci.yml)**: Create the workflow → all 28 workflow tests should pass
4. **Task 6 (dev deps)**: Add dev extras to packages → remaining 9 dev dependency tests pass

---

## Files Generated

| File | Purpose | AC |
|------|---------|-----|
| `eusolicit-app/tests/unit/test_ci_ruff_config.py` | Validates ruff.toml: existence, rule sets (I/E/W/F/UP), target-version, line-length, exclusions, per-file-ignores | AC 4 |
| `eusolicit-app/tests/unit/test_ci_mypy_config.py` | Validates [tool.mypy] in pyproject.toml: existence, python_version, warning flags, per-package overrides | AC 5 |
| `eusolicit-app/tests/unit/test_ci_workflow_structure.py` | Validates ci.yml: triggers, 8-job matrix, ruff/mypy/pytest steps, pip caching, Docker caching, failure reporting, performance | AC 1, 2, 3, 6, 7, 8, 9 |
| `eusolicit-app/tests/unit/test_ci_dev_dependencies.py` | Validates dev extras in all 8 project pyproject.toml files: pytest, pytest-cov, ruff, mypy | AC 3, 6 |
| `eusolicit-docs/test-artifacts/atdd-checklist-1-8-github-actions-ci-pipeline.md` | This ATDD checklist | -- |

---

## Next Steps (TDD Green Phase)

After implementing the configuration files:

1. **Task 1 — ruff.toml**: Create at repo root
   - Run: `cd eusolicit-app && python -m pytest tests/unit/test_ci_ruff_config.py -v`
   - Expected: 13/13 pass

2. **Task 2 — mypy config**: Add `[tool.mypy]` to pyproject.toml
   - Run: `cd eusolicit-app && python -m pytest tests/unit/test_ci_mypy_config.py -v`
   - Expected: 7/7 pass

3. **Task 3 — ci.yml**: Create `.github/workflows/ci.yml`
   - Run: `cd eusolicit-app && python -m pytest tests/unit/test_ci_workflow_structure.py -v`
   - Expected: 28/28 pass

4. **Task 6 — dev deps**: Add dev extras to shared packages
   - Run: `cd eusolicit-app && python -m pytest tests/unit/test_ci_dev_dependencies.py -v`
   - Expected: 30/30 pass (currently 21/30)

5. **Full regression**: `cd eusolicit-app && python -m pytest tests/unit/test_ci_*.py -v`
   - Expected: 78/78 pass

6. **Verify no regressions**: `make test-unit` — all existing tests still pass

### Implementation Order (Recommended)

Based on dependency analysis:

1. **ruff.toml** (Task 1, AC 4) — standalone, no dependencies
2. **[tool.mypy] in pyproject.toml** (Task 2, AC 5) — standalone, extends existing file
3. **Dev extras in package pyproject.toml files** (Task 6, AC 3) — eusolicit-common needs full section; models/kraftdata need ruff+mypy added
4. **ci.yml** (Task 3, AC 1-3, 6-9) — depends on ruff.toml and mypy config existing for CI steps to work
5. **Verify ruff/mypy pass locally** (Task 7.1, 7.2) — run with new configs before CI creation
6. **Docker layer caching** (Task 5, AC 7) — optional/deferred, evaluate after ci.yml works

---

## Quality Gate Criteria

| Gate | Threshold | Status |
|------|-----------|--------|
| P0 tests pass rate | 100% | Pending implementation |
| P1 tests pass rate | >= 95% | Pending implementation |
| P2 tests pass rate | >= 90% | Pending implementation |
| No high-risk items unmitigated | E01-R-006 covered | ✅ |
| Existing test regression | 0 new failures | Verify after implementation |
| ruff check passes locally | 0 errors | Pending implementation |
| mypy passes locally | 0 errors | Pending implementation |
| ci.yml valid YAML | No syntax errors | Pending implementation |

---

## Validation Checklist

- [x] Prerequisites satisfied (story approved, pytest configured, dev env available)
- [x] All 4 test files created with correct structure and markers
- [x] Tests cover all 9 acceptance criteria
- [x] Tests designed to fail before implementation (ATDD RED phase)
- [x] Priority tags assigned per epic-level test design
- [x] No placeholder assertions (all assert expected behavior)
- [x] Test patterns match existing project conventions (class-based, docstrings, pytestmark)
- [x] 57 tests confirmed FAILING, 21 confirmed PASSING (pre-existing dev extras)
- [x] Checklist saved to `test-artifacts/` directory
- [x] No DO NOT TOUCH files modified
- [x] ruff.toml tests validate all 5 required rule sets (I, E, W, F, UP)
- [x] mypy tests validate per-package overrides and test file relaxation
- [x] ci.yml tests validate all 8 matrix entries by project name
- [x] ci.yml tests validate 3 per-job steps (ruff, mypy, pytest)
- [x] Dev dependency tests use parametrize for all 8 projects
- [x] TOML parsing uses stdlib `tomllib` (no external dependency)
- [x] YAML parsing uses `pyyaml` (commonly available)

---

**Generated by:** BMad TEA Agent -- ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
