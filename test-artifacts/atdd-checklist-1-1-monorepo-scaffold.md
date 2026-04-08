---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-06'
workflowType: 'testarch-atdd'
storyId: '1-1-monorepo-scaffold-project-structure'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/1-1-monorepo-scaffold-project-structure.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-01.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
---

# ATDD Checklist: Story 1.1 — Monorepo Scaffold & Project Structure

## Story Summary

**Epic:** E01 — Infrastructure & Monorepo Foundation
**Story:** 1.1 — Monorepo Scaffold & Project Structure
**Sprint:** 1 | **Priority:** P0 (scaffold is prerequisite for all Sprint 2+ work)
**Risk-Driven:** Yes — E01-R-005 (shared package editable install breakage, Score: 4)

**Focus Areas:**
- Root workspace configuration referencing all 5 services and 3 shared packages (AC1)
- Standardized service directory structure with src/ layout (AC2, AC3)
- Shared package scaffolds with version exports (AC4)
- Frontend Next.js placeholder (AC5)
- Infrastructure directory placeholders (AC6)
- Shared package imports work from all services via editable installs (AC7) — **P0**
- `pip install -e .` succeeds in every service directory (AC8) — **P0**

---

## TDD Red Phase (Current)

**Status:** RED — All tests use `@pytest.mark.skip(reason="ATDD RED: scaffold not implemented — Story 1.1")` and will be skipped until features are implemented.

| Test Class | File | Test Functions | Expanded Tests | AC | Status |
|------------|------|---------------|----------------|----|----|
| `TestAC1RootWorkspaceConfig` | `tests/smoke/test_scaffold_story_1_1.py` | 3 | 3 | AC1 | Skipped (RED) |
| `TestAC2ServiceDirectoryContents` | `tests/smoke/test_scaffold_story_1_1.py` | 5 | 25 | AC2 | Skipped (RED) |
| `TestAC3ServicesScaffolded` | `tests/smoke/test_scaffold_story_1_1.py` | 3 | 15 | AC3 | Skipped (RED) |
| `TestAC4SharedPackagesScaffolded` | `tests/smoke/test_scaffold_story_1_1.py` | 6 | 18 | AC4 | Skipped (RED) |
| `TestAC5FrontendPlaceholder` | `tests/smoke/test_scaffold_story_1_1.py` | 6 | 6 | AC5 | Skipped (RED) |
| `TestAC6InfraDirectories` | `tests/smoke/test_scaffold_story_1_1.py` | 4 | 4 | AC6 | Skipped (RED) |
| `TestAC7SharedPackageImports` | `tests/smoke/test_scaffold_story_1_1.py` | 3 | 15 | AC7 | Skipped (RED) |
| `TestAC8EditableInstall` | `tests/smoke/test_scaffold_story_1_1.py` | 2 | 8 | AC8 | Skipped (RED) |
| **Total** | **1 file** | **32 functions** | **94 test cases** | **AC1–AC8** | **All skipped** |

---

## Acceptance Criteria Coverage

### AC1: Root pyproject.toml Workspace Config

| AC Detail | Test | Priority |
|-----------|------|----------|
| Root pyproject.toml references all 5 services | `test_root_pyproject_references_all_services` | P0 |
| Root pyproject.toml references all 3 shared packages | `test_root_pyproject_references_all_shared_packages` | P0 |
| Existing pytest/coverage config preserved after workspace extension | `test_root_pyproject_preserves_existing_pytest_config` | P0 |

### AC2: Service Directory Contents

| AC Detail | Test (×5 services) | Priority |
|-----------|---------------------|----------|
| Each service has pyproject.toml | `test_service_has_pyproject_toml[<service>]` | P0 |
| Each service has Dockerfile | `test_service_has_dockerfile[<service>]` | P0 |
| Each service has alembic.ini | `test_service_has_alembic_ini[<service>]` | P0 |
| Each service has src/<module>/__init__.py | `test_service_has_src_layout[<service>]` | P0 |
| Each service has tests/ directory | `test_service_has_tests_directory[<service>]` | P0 |

### AC3: All 5 Services Scaffolded

| AC Detail | Test (×5 services) | Priority |
|-----------|---------------------|----------|
| Service directory exists | `test_service_directory_exists[<service>]` | P0 |
| pyproject.toml declares correct project name and src/ layout | `test_service_pyproject_declares_correct_name[<service>-<module>]` | P0 |
| pyproject.toml declares all 3 shared package dependencies | `test_service_pyproject_declares_shared_package_deps[<service>]` | P0 |

### AC4: All 3 Shared Packages Scaffolded

| AC Detail | Test (×3 packages) | Priority |
|-----------|---------------------|----------|
| Package directory exists | `test_shared_package_directory_exists[<pkg>]` | P0 |
| Package has pyproject.toml | `test_shared_package_has_pyproject_toml[<pkg>]` | P0 |
| Package has src/<module>/__init__.py | `test_shared_package_has_init_py[<pkg>]` | P0 |
| Package __init__.py exports `__version__ = "0.1.0"` | `test_shared_package_exports_version[<pkg>]` | P0 |
| Package pyproject.toml uses src/ layout | `test_shared_package_pyproject_uses_src_layout[<pkg>]` | P0 |
| Package pyproject.toml declares correct name | `test_shared_package_pyproject_declares_correct_name[<pkg>]` | P0 |

### AC5: Frontend Next.js Placeholder

| AC Detail | Test | Priority |
|-----------|------|----------|
| frontend/ directory exists | `test_frontend_directory_exists` | P1 |
| frontend/package.json exists | `test_frontend_has_package_json` | P1 |
| frontend/tsconfig.json exists | `test_frontend_has_tsconfig` | P1 |
| next.config.{js,mjs,ts} exists | `test_frontend_has_next_config` | P1 |
| App router page or pages/index exists | `test_frontend_has_placeholder_page` | P1 |
| package.json declares next dependency | `test_frontend_package_json_has_next_dependency` | P1 |

### AC6: Infrastructure Directory Placeholders

| AC Detail | Test | Priority |
|-----------|------|----------|
| infra/helm/README.md exists | `test_infra_helm_readme_exists` | P2 |
| infra/terraform/README.md exists | `test_infra_terraform_readme_exists` | P2 |
| infra/helm/README.md is not empty | `test_infra_helm_readme_not_empty` | P2 |
| infra/terraform/README.md is not empty | `test_infra_terraform_readme_not_empty` | P2 |

### AC7: Shared Package Imports (P0 — E01-R-005)

| AC Detail | Test (×5 services) | Priority |
|-----------|---------------------|----------|
| Service imports eusolicit_common | `test_service_can_import_eusolicit_common[<service>]` | **P0** |
| Service imports eusolicit_models | `test_service_can_import_eusolicit_models[<service>]` | **P0** |
| Service imports eusolicit_kraftdata | `test_service_can_import_eusolicit_kraftdata[<service>]` | **P0** |

### AC8: pip install -e . Succeeds (P0 — E01-R-005)

| AC Detail | Test (×5 services + ×3 packages) | Priority |
|-----------|-----------------------------------|----------|
| pip install -e . succeeds per service | `test_pip_install_editable_succeeds[<service>]` | **P0** |
| pip install -e . succeeds per shared package | `test_pip_install_editable_succeeds_for_package[<pkg>]` | **P0** |

---

## Epic Test Design Traceability

| Epic Test Design Requirement | AC | Test Class(es) | Priority |
|------------------------------|-----|----------------|----------|
| P0: All 5 services import all 3 shared packages | AC7, AC8 | `TestAC7SharedPackageImports`, `TestAC8EditableInstall` | **P0** |
| P0: CI pipeline lint + type check + test green for all 8 projects | AC1, AC3 | `TestAC1RootWorkspaceConfig`, `TestAC3ServicesScaffolded` | **P0** |
| E01-R-005: Shared package editable install breakage (Score: 4) | AC7, AC8 | `TestAC7SharedPackageImports`, `TestAC8EditableInstall` | **P0** |

---

## Fixture Needs

_None required for GREEN phase._ These tests are pure filesystem and subprocess assertions — no database, Redis, or service fixtures needed.

| Resource | Purpose | Status |
|----------|---------|--------|
| Python 3.12+ runtime | Subprocess import checks, pip install verification | Available |
| `tomllib` (stdlib) | Parse pyproject.toml files | Available (Python 3.12+) |
| File system access | Verify directory/file existence | Available |

---

## Mock Requirements

_None._ Story 1.1 tests are structural scaffold verification — no services, databases, or external systems involved.

---

## Required data-testid Attributes

_Not applicable — all tests are filesystem and import verification, no UI._

---

## Implementation Guidance

### What Must Be Created (to Turn Tests GREEN)

**Shared Packages (AC4, AC7, AC8):**
- `packages/eusolicit-common/pyproject.toml` — name: `eusolicit-common`, src/ layout, deps: fastapi, structlog, pydantic, pydantic-settings, redis
- `packages/eusolicit-common/src/eusolicit_common/__init__.py` — exports `__version__ = "0.1.0"`
- `packages/eusolicit-models/pyproject.toml` — name: `eusolicit-models`, src/ layout, deps: pydantic
- `packages/eusolicit-models/src/eusolicit_models/__init__.py` — exports `__version__ = "0.1.0"`
- `packages/eusolicit-kraftdata/pyproject.toml` — name: `eusolicit-kraftdata`, src/ layout, deps: pydantic
- `packages/eusolicit-kraftdata/src/eusolicit_kraftdata/__init__.py` — exports `__version__ = "0.1.0"`

**Services (AC2, AC3, AC7, AC8):**

For each of: `client-api`, `admin-api`, `data-pipeline`, `ai-gateway`, `notification`:
- `services/<svc>/pyproject.toml` — correct name, src/ layout, all 3 shared packages in deps
- `services/<svc>/src/<module>/__init__.py` — module init
- `services/<svc>/Dockerfile` — multi-stage build
- `services/<svc>/alembic.ini` — config stub
- `services/<svc>/tests/` — already exists (DO NOT TOUCH)

**Root Config (AC1):**
- Extend `pyproject.toml` with workspace references to all 5 services and 3 packages
- Preserve existing `[tool.pytest.ini_options]` and `[tool.coverage.*]` sections

**Frontend (AC5):**
- `frontend/package.json` — declares `next` dependency (14+)
- `frontend/tsconfig.json`
- `frontend/next.config.{js,mjs,ts}`
- `frontend/app/page.tsx` — minimal placeholder

**Infrastructure (AC6):**
- `infra/helm/README.md` — non-empty placeholder
- `infra/terraform/README.md` — non-empty placeholder

### Service-Specific Dependencies (from story spec)

| Service | Additional Dependencies |
|---------|------------------------|
| client-api | httpx, pyjwt, stripe, python-multipart |
| admin-api | httpx, pyjwt |
| data-pipeline | celery[redis], httpx |
| ai-gateway | httpx, sse-starlette |
| notification | celery[redis], httpx, sendgrid |

---

## Red-Green-Refactor Workflow

### RED Phase (Complete — TEA)

- [x] 94 acceptance test cases generated (32 test functions, many parametrized)
- [x] All tests use `@pytest.mark.skip` to document intentional RED phase
- [x] Tests assert EXPECTED behavior against unimplemented scaffold
- [x] Tests organized by acceptance criterion (AC1–AC8) in dedicated classes
- [x] P0 tests for AC7/AC8 linked to E01-R-005 risk mitigation
- [x] Test file: `eusolicit-app/tests/smoke/test_scaffold_story_1_1.py`

### GREEN Phase (Next — DEV Team)

After implementing the scaffold:

1. Remove `@pytest.mark.skip` from test classes one AC at a time
2. Run tests for each AC group:
   ```bash
   # Run all Story 1.1 tests
   cd eusolicit-app
   python -m pytest tests/smoke/test_scaffold_story_1_1.py -v

   # Run specific AC group
   python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC4"

   # Run P0 critical tests (imports + editable install)
   python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC7 or TestAC8"
   ```
3. Verify tests PASS (green phase)
4. If any tests fail:
   - **Implementation bug**: Fix the scaffold file/config
   - **Test bug**: Fix the test assertion (update this checklist)
5. Commit passing tests

### REFACTOR Phase (DEV Team)

- Consolidate parametrized assertions if test count feels excessive
- Add pyproject.toml dependency version validation if needed
- Consider adding Dockerfile syntax validation (hadolint) as P2

---

## Execution Commands

```bash
# Navigate to app root
cd eusolicit-app

# Run ALL Story 1.1 tests (currently all skipped)
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v

# Run by AC group
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC1"
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC2"
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC3"
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC4"
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC5"
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC6"
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC7"
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC8"

# Run P0 only (imports + editable install)
python -m pytest tests/smoke/test_scaffold_story_1_1.py -v -k "TestAC7 or TestAC8"

# Run smoke marker (includes these + other smoke tests)
python -m pytest -m smoke -v
```

---

## Completion Summary

| Metric | Value |
|--------|-------|
| **Story ID** | 1-1-monorepo-scaffold-project-structure |
| **Primary test level** | Unit / Smoke (filesystem + subprocess assertions) |
| **Test functions** | 32 |
| **Expanded test cases** | 94 (via parametrization across 5 services, 3 packages) |
| **Test files** | 1 (`tests/smoke/test_scaffold_story_1_1.py`) |
| **Fixtures needed** | 0 (pure filesystem/subprocess) |
| **Mock requirements** | 0 |
| **data-testid requirements** | 0 |
| **External dependencies** | None (Python stdlib only + pytest) |
| **Execution mode** | Parallel-safe (no shared state) |
| **TDD phase** | RED (all tests skipped) |

### Priority Breakdown

| Priority | Test Cases | AC Coverage |
|----------|------------|-------------|
| **P0** | 69 | AC1, AC2, AC3, AC4, AC7, AC8 |
| **P1** | 6 | AC5 (frontend placeholder) |
| **P2** | 4 | AC6 (infra directories) |
| **Total** | **94** | **AC1–AC8 (100%)** |

### Risk Mitigation Coverage

| Risk ID | Score | Mitigation | Test Coverage |
|---------|-------|------------|---------------|
| E01-R-005 | 4 | Test `pip install -e .` both locally; verify all 3 packages importable from each service | AC7 (15 tests) + AC8 (8 tests) = 23 test cases |

### Knowledge Base References Applied

- `test-levels-framework.md` — Smoke-level test selection for structural verification
- `test-priorities-matrix.md` — P0 criteria (blocks all Sprint 2+ development)
- `test-quality.md` — No external dependencies, deterministic assertions, parallel-safe

### Next Steps

1. **Implement Story 1.1** — Create all scaffold files per the story spec
2. **Remove `@pytest.mark.skip`** — One AC group at a time, verify GREEN phase
3. **Run full test suite** — `python -m pytest tests/smoke/test_scaffold_story_1_1.py -v`
4. **Verify CI integration** — Tests should run as part of `make test` / CI pipeline
5. **Proceed to Story 1.2** — Docker Compose environment (next scaffold story)

---

**Generated by:** BMad TEA Agent — ATDD Module
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
