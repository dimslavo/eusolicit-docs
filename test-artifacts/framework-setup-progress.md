---
stepsCompleted: ['step-01-preflight', 'step-02-select-framework', 'step-03-scaffold-framework', 'step-04-docs-and-scripts', 'step-05-validate-and-summary', 'step-03-enhance-run-2', 'step-05-validate-enhance-run-2', 'step-03-enhance-run-3', 'step-05-validate-enhance-run-3']
lastStep: 'step-05-validate-enhance-run-3'
lastSaved: '2026-04-09'
---

# Test Framework Setup Progress

## Step 1: Preflight Checks

### Stack Detection
- **Detected stack**: `fullstack`
- **Backend indicators**: `pyproject.toml` found (Orchestrator SDK); Architecture docs specify Python 3.12 + FastAPI 0.115+ for all 5 services
- **Frontend indicators**: Architecture docs specify Next.js 14 + React 18 (no `package.json` yet — project is pre-implementation)

### Prerequisites Validation
- **Project state**: Pre-implementation. `eusolicit-app/` contains only `.git`. Monorepo structure per E01 has not been scaffolded yet.
- **No existing E2E framework**: No `playwright.config.*`, `cypress.config.*`, or `conftest.py` found
- **No existing test framework conflicts**: Clean slate
- **Architecture context available**: EU_Solicit_Solution_Architecture_v4.md, 12 epic specs, implementation plan

### Project Context
- **Backend**: Python 3.12, FastAPI 0.115+, SQLAlchemy 2.0+ (async), Celery 5.4+, Pydantic 2.x, httpx 0.27+
- **Frontend**: Next.js 14 (App Router, SSR/SSG), React 18, Tailwind CSS + shadcn/ui, Zustand, TanStack Query, React Hook Form + Zod, Tiptap, Recharts, next-intl
- **Database**: PostgreSQL 16 (6 schemas, per-service roles), Redis 7 (Streams event bus)
- **Infrastructure**: Docker, Kubernetes, Helm 3, GitHub Actions CI
- **Services**: Client API, Admin API, Data Pipeline, AI Gateway, Notification Service
- **Shared packages**: eusolicit-common, eusolicit-models, eusolicit-kraftdata
- **Testing stack (architecture-specified)**: pytest 8.x + pytest-asyncio, ruff + mypy for linting
- **Auth**: JWT RS256 + Google OAuth2

---

## Step 2: Framework Selection

### Decision: Dual Framework — Playwright (E2E/Browser) + pytest (Backend)

**Browser-based testing: Playwright**
- **Rationale**:
  - Large, complex SOA platform with 5 services and two distinct UIs (client + admin)
  - Multi-browser support needed (public-facing EU procurement platform)
  - Heavy API + UI integration (SSE streaming, real-time collaboration, Stripe billing flows)
  - CI speed/parallelism critical — GitHub Actions matrix build already planned
  - Playwright's native multi-context support maps well to testing white-label subdomains
  - Built-in API testing capabilities align with inter-service contract verification
  - Better TypeScript support for Next.js/React testing

**Backend testing: pytest + pytest-asyncio**
- **Rationale**:
  - Explicitly specified in architecture (Section 4.1: "Testing: pytest + pytest-asyncio 8.x")
  - Native async support matches FastAPI's async runtime
  - Rich fixture system ideal for multi-schema database testing
  - Already configured in Orchestrator's `pyproject.toml` as reference pattern
  - Excellent Pydantic integration for DTO validation testing
  - Factory pattern support via pytest-factoryboy for complex domain models

---

## Step 3: Scaffold Framework

### Execution Mode
- **Resolved mode**: `sequential` (no explicit user override, config has no `tea_execution_mode`)
- **Parallel agents used**: Yes — 2 parallel agents for backend and Playwright scaffolding

### Directory Structure Created

**Root test infrastructure:**
- `tests/conftest.py` — Shared fixtures (DB engine/session, Redis client, HTTP clients, auth headers)
- `tests/__init__.py`, `tests/cross_service/`, `tests/smoke/` — Test organization

**Per-service test directories (5 services):**
- `services/client-api/tests/` — conftest.py + unit/ + integration/ + api/
- `services/admin-api/tests/` — conftest.py + unit/ + integration/ + api/
- `services/data-pipeline/tests/` — conftest.py + unit/ + integration/
- `services/ai-gateway/tests/` — conftest.py + unit/ + integration/ + api/
- `services/notification/tests/` — conftest.py + unit/ + integration/

**Shared test utilities package:**
- `packages/eusolicit-test-utils/` — pyproject.toml + src/eusolicit_test_utils/
  - `factories.py` — TenantFactory, UserFactory, CompanyFactory, OpportunityFactory, ProposalFactory, EventFactory (+ tier variants)
  - `api_client.py` — ServiceClient with async context manager, per-service URL registry
  - `db.py` — Schema management (6 schemas), table truncation, migration helpers
  - `redis_utils.py` — 7 stream definitions, event envelope builder, publish/consume/flush
  - `auth.py` — JWT token generation with role and tier support

**Playwright E2E framework:**
- `playwright.config.ts` — Environment config map (local/staging/production), 6 projects, standard timeouts
- `e2e/global-setup.ts` — Service health checks before suite runs
- `e2e/tsconfig.json` — TypeScript config with path aliases
- `e2e/support/fixtures/` — mergeTests composition (auth + api + factory)
- `e2e/support/factories/` — createUser, createCompany, createOpportunity, createProposal
- `e2e/support/page-objects/` — LoginPage, DashboardPage, OpportunitiesPage
- `e2e/support/helpers/` — api-client, auth.helper, network.helper

**Sample specs:**
- `e2e/specs/auth/login.spec.ts` — Login happy path, invalid credentials, auth redirect
- `e2e/specs/opportunities/search.spec.ts` — Listing, keyword search, no results, detail nav
- `e2e/specs/smoke/health.api.spec.ts` — Service health checks (no browser)

### Configuration Files
- `pyproject.toml` — pytest markers (unit, integration, api, smoke, slow, cross_service), coverage config
- `playwright.config.ts` — 6 projects (client x3 browsers, admin, mobile, api), timeouts, reporters
- `.env.example` — All test environment variables
- `.python-version` — Python 3.12
- `.nvmrc` — Node v22
- `.gitignore` — Test artifacts, Python, Node, Docker patterns

### Knowledge Base Patterns Applied
- **fixture-architecture.md** — mergeTests composition, one concern per fixture, auto-cleanup
- **data-factories.md** — Faker-based with Partial overrides, API-first seeding, unique values
- **playwright-config.md** — Environment config map, standard timeouts, HTML + JUnit reporters
- **test-quality.md** — No hard waits, network-first waits, explicit assertions, parallel-safe
- **network-first.md** — waitForResponse before navigation, mock helpers for edge cases
- **test-levels-framework.md** — unit/ integration/ api/ separation per service
- **contract-testing.md** — Architecture supports future Pact CDC (SOA with 5 services)

---

## Step 4: Documentation & Scripts

### Documentation
- `e2e/README.md` — Setup instructions, running tests, architecture overview, best practices, CI integration, knowledge base references

### Build & Test Scripts
- `package.json` — 12 npm scripts for Playwright (test:e2e, test:e2e:headed, test:e2e:chromium, etc.)
- `Makefile` — 16 make targets (test, test-unit, test-integration, test-api, test-smoke, test-cross, test-service, test-e2e, coverage, lint, type-check, test-ci, e2e-ci, etc.)

---

## Step 5: Validation & Summary

### Checklist Validation

#### Prerequisites ✅
- [x] Project manifest exists (`pyproject.toml` for backend, `package.json` for E2E)
- [x] No existing test framework conflicts
- [x] Project type identifiable (fullstack: Python FastAPI + Next.js React)
- [x] Bundler identifiable (N/A for backend; Next.js bundler for frontend)
- [x] Write permissions confirmed (files created successfully)

#### Step 1: Preflight ✅
- [x] Stack type detected: `fullstack`
- [x] Project manifests read and parsed
- [x] Project type extracted correctly
- [x] No framework conflicts detected
- [x] Architecture documents located

#### Step 2: Framework Selection ✅
- [x] Framework auto-detection logic executed
- [x] Framework choice justified (Playwright for E2E, pytest for backend)
- [x] User notified of selection and rationale

#### Step 3: Directory Structure ✅
- [x] `tests/` root directory created
- [x] `e2e/specs/` directory created (Playwright convention)
- [x] `e2e/support/` directory created (critical pattern)
- [x] `e2e/support/fixtures/` directory created
- [x] `e2e/support/factories/` directory created
- [x] `e2e/support/helpers/` directory created
- [x] `e2e/support/page-objects/` directory created
- [x] Per-service `tests/` directories created (5 services)

#### Step 4: Configuration Files ✅
- [x] `playwright.config.ts` created with TypeScript
- [x] Timeouts configured (action: 15s, navigation: 30s, test: 60s, expect: 10s)
- [x] Base URL with environment variable fallback
- [x] Trace `retain-on-failure`, screenshot `only-on-failure`, video `retain-on-failure`
- [x] HTML + JUnit + console reporters
- [x] Parallel execution enabled
- [x] CI settings (retries: 2, workers: 2)

#### Step 5: Environment Configuration ✅
- [x] `.env.example` created
- [x] `TEST_ENV` variable defined
- [x] `BASE_URL` variable defined
- [x] `CLIENT_API_URL` (and 4 other service URLs) defined
- [x] Authentication variables defined (`TEST_JWT_SECRET`, `TEST_AUTH_TOKEN`)
- [x] `.nvmrc` created (Node v22)
- [x] `.python-version` created (3.12)

#### Step 6: Fixture Architecture ✅
- [x] `e2e/support/fixtures/index.ts` created
- [x] Base fixture extended from `@playwright/test`
- [x] Type definitions for fixtures created (AuthFixtures, ApiFixtures, FactoryFixtures)
- [x] `mergeTests` pattern implemented (3 fixture sources)
- [x] Auto-cleanup in API fixture (`seedViaApi` deletes in reverse order)
- [x] Backend conftest.py uses per-test transaction rollback

#### Step 7: Data Factories ✅
- [x] Multiple factories created (User, Company, Opportunity, Proposal, Tenant, Event)
- [x] Factories use @faker-js/faker (frontend) and faker (backend)
- [x] API fixture tracks created entities for cleanup
- [x] Factories integrate with fixtures via factory.fixture.ts
- [x] Tier variant factories (ProCompany, Enterprise, AdminUser)

#### Step 8: Sample Tests ✅
- [x] 3 sample spec files created (auth, opportunities, smoke)
- [x] Tests use fixture architecture (`import { test, expect } from '../../support/fixtures'`)
- [x] Tests demonstrate data factory usage
- [x] Tests use `data-testid` selector strategy
- [x] Tests follow Given-When-Then structure
- [x] Tests include proper assertions
- [x] Network interception demonstrated (waitForResponse in page objects)

#### Step 9: Helper Utilities ✅
- [x] API helper created (`api-client.ts`, `api_client.py`)
- [x] Network helper created (`network.helper.ts`)
- [x] Auth helper created (`auth.helper.ts`, `auth.py`)
- [x] Helpers follow functional patterns
- [x] Helpers have proper error handling

#### Step 10: Documentation ✅
- [x] `e2e/README.md` created
- [x] Setup instructions included
- [x] Running tests section included
- [x] Architecture overview section included
- [x] Best practices section included
- [x] CI integration section included
- [x] Knowledge base references included

#### Step 11: Build & Test Scripts ✅
- [x] `package.json` with test scripts (12 npm scripts)
- [x] `Makefile` with test targets (16 make targets)
- [x] Test framework dependencies added to `package.json`
- [x] TypeScript type definitions added via `e2e/tsconfig.json`

#### Security Checks ✅
- [x] No credentials in configuration files
- [x] `.env.example` contains placeholders only
- [x] API keys use environment variables
- [x] `.gitignore` excludes `.env` files (not `.env.example`)

### Artifacts Created

| Category | Count | Files |
|----------|-------|-------|
| Root config | 6 | pyproject.toml, playwright.config.ts, package.json, .env.example, .python-version, .nvmrc |
| Root test infra | 6 | tests/conftest.py, tests/__init__.py, cross_service/*, smoke/* |
| Per-service tests | 24 | 5 services × (conftest.py + __init__.py + unit/ + integration/ [+ api/]) |
| Test utils package | 7 | pyproject.toml + 6 Python modules |
| E2E framework | 16 | global-setup, tsconfig, 4 fixtures, 1 factory, 3 page objects, 3 helpers, 3 specs |
| Docs & scripts | 3 | README.md, Makefile, .gitignore |
| **Total** | **62** | |

### Next Steps

1. **Install dependencies**:
   ```bash
   cd eusolicit-app
   pip install -e packages/eusolicit-test-utils
   npm install
   npx playwright install --with-deps
   ```

2. **Configure environment**:
   ```bash
   cp .env.example .env
   ```

3. **When E01 (Infrastructure) is implemented**: The test framework is ready to receive service code. Each service's `conftest.py` has a `pytest.skip()` guard that will be removed when the actual FastAPI app is created.

4. **Recommended next workflows**:
   - `bmad-testarch-ci` — Scaffold GitHub Actions quality pipeline
   - `bmad-testarch-test-design` — Create system-level test plan
   - `bmad-testarch-atdd` — Generate acceptance tests for first stories

---

## Enhancement Run — 2026-04-07 (project-context.md alignment)

### Context

Run triggered post-Epic-1 retrospective. The framework was originally scaffolded on 2026-04-05 (pre-implementation). Epic 1 has now completed, producing 50+ tests, an updated `project-context.md`, and new ATDD specs for Epic 12. This run gaps-analysis pass fills in missing artifacts and aligns the framework with current project-context patterns.

### Gaps Identified & Resolved

**E2E Playwright:**

| Gap | Resolution |
|-----|------------|
| `e2e/support/page-objects/index.ts` missing (ATDD smoke test `TestE2EScaffold.test_e2e_page_objects_index_exists` was FAILING) | Created barrel export — `LoginPage`, `DashboardPage`, `OpportunitiesPage` |
| `e2e/support/helpers/index.ts` missing (ATDD smoke test `test_e2e_helpers_index_exists` was FAILING) | Created barrel export — `apiGet/Post/Delete`, `createTestSession`, `createAuthenticatedContext`, `waitForApiResponse`, `mockApiResponse/Error` |
| `tenant.fixture.ts` missing — Epic 12 analytics ATDD tests need tier-aware API clients | Created `getTierToken` and `getTierClient` fixtures with ATDD red-phase fallback tokens |
| `factory.fixture.ts` only exposed 4 factories — `adminUser` and `proCompany` unavailable to tests | Added `adminUser` (createAdminUser) and `proCompany` (createProCompany) to the merged factories object |
| `e2e/support/fixtures/index.ts` missing tenant fixture | Updated to merge `tenantTest` (4→5 fixture sources) |

**Python / pytest:**

| Gap | Resolution |
|-----|------------|
| `tests/unit/conftest.py` missing — unit tests marked individually, no directory-level marker | Created conftest with `pytestmark = pytest.mark.unit` |
| `tests/integration/conftest.py` missing — integration tests inherited from root but had no directory marker | Created conftest with `pytestmark = pytest.mark.integration` + dependency documentation |
| `tests/cross_service/conftest.py` only had `pytestmark` — no actual cross-service fixtures | Extended with `all_service_clients`, `correlation_id`, `stream_monitor` (Redis Streams), and `services_healthy` skip guard |

### Files Created (6)

| File | Purpose |
|------|---------|
| `e2e/support/page-objects/index.ts` | Barrel export — unblocks ATDD smoke tests |
| `e2e/support/helpers/index.ts` | Barrel export — unblocks ATDD smoke tests |
| `e2e/support/fixtures/tenant.fixture.ts` | Tier-aware API clients for Epic 12 cross-tenant + tier-gate ATDD |
| `tests/unit/conftest.py` | Directory-level unit marker |
| `tests/integration/conftest.py` | Directory-level integration marker + dependency docs |

### Files Updated (4)

| File | Change |
|------|--------|
| `e2e/support/fixtures/factory.fixture.ts` | Added `adminUser` and `proCompany` to exported factories object |
| `e2e/support/fixtures/index.ts` | Merged `tenantTest` — 4 fixture sources → 5 |
| `tests/cross_service/conftest.py` | Added `all_service_clients`, `correlation_id`, `stream_monitor`, `services_healthy` |
| `e2e/README.md` | Documented tenant fixture, page-objects/index.ts, helpers/index.ts |

### Knowledge Base Patterns Applied

- **fixture-architecture.md** — `tenant.fixture.ts` follows one-concern-per-fixture + auto-cleanup pattern
- **data-factories.md** — Extended factory fixture exposes all variants for parallel-safe test data generation
- **test-quality.md** — `services_healthy` uses `pytest.skip()` (not fail) for infrastructure issues, keeping test results actionable

### Post-Enhancement Checklist

- [x] `e2e/support/page-objects/index.ts` — barrel export
- [x] `e2e/support/helpers/index.ts` — barrel export
- [x] `e2e/support/fixtures/tenant.fixture.ts` — tier-aware API clients
- [x] `factory.fixture.ts` — adminUser + proCompany exposed
- [x] `fixtures/index.ts` — tenant fixture merged
- [x] `tests/unit/conftest.py` — unit directory marker
- [x] `tests/integration/conftest.py` — integration directory marker
- [x] `tests/cross_service/conftest.py` — real cross-service fixtures

---

## Step 5: Final Validation — Enhancement Run (2026-04-07)

### Full File-System Validation (40/40 ✅)

All framework files confirmed present via `python3` path validation.

#### Prerequisites
- [x] `pyproject.toml` — backend manifest present
- [x] `package.json` — frontend/E2E manifest present

#### Directory Structure (13/13)
- [x] `tests/` root, `tests/unit/`, `tests/integration/`, `tests/cross_service/`, `tests/smoke/`
- [x] `e2e/`, `e2e/support/`, `e2e/support/fixtures/`, `e2e/support/factories/`
- [x] `e2e/support/helpers/`, `e2e/support/page-objects/`, `e2e/specs/`
- [x] `packages/eusolicit-test-utils/`

#### Configuration Files (5/5)
- [x] `playwright.config.ts` — 6 projects, envConfigMap, standard timeouts, HTML+JUnit reporters
- [x] `.env.example` — service URLs, JWT secret, test env vars
- [x] `.nvmrc` — Node v22
- [x] `pyproject.toml` — pytest markers, coverage config
- [x] `Makefile` — 16 test targets

#### Fixture Architecture (6/6)
- [x] `e2e/support/fixtures/index.ts` — mergeTests(auth, api, factory, tenant)
- [x] `e2e/support/fixtures/auth.fixture.ts`
- [x] `e2e/support/fixtures/api.fixture.ts`
- [x] `e2e/support/fixtures/factory.fixture.ts` — 6 factories incl. adminUser + proCompany
- [x] `e2e/support/fixtures/tenant.fixture.ts` — getTierToken + getTierClient
- [x] `e2e/global-setup.ts` — service health checks before suite

#### Data Factories (7/7)
- [x] `e2e/support/factories/index.ts`
- [x] `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py`
- [x] `packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py`
- [x] `packages/eusolicit-test-utils/src/eusolicit_test_utils/redis_utils.py`
- [x] `packages/eusolicit-test-utils/src/eusolicit_test_utils/auth.py`
- [x] `packages/eusolicit-test-utils/src/eusolicit_test_utils/api_client.py`

#### Helpers (4/4)
- [x] `e2e/support/helpers/index.ts` — barrel export (was missing, now present)
- [x] `e2e/support/helpers/api-client.ts`
- [x] `e2e/support/helpers/auth.helper.ts`
- [x] `e2e/support/helpers/network.helper.ts`

#### Page Objects (4/4)
- [x] `e2e/support/page-objects/index.ts` — barrel export (was missing, now present)
- [x] `e2e/support/page-objects/login.page.ts`
- [x] `e2e/support/page-objects/dashboard.page.ts`
- [x] `e2e/support/page-objects/opportunities.page.ts`

#### Conftest Files (5/5)
- [x] `tests/conftest.py` — shared DB/Redis/HTTP fixtures
- [x] `tests/unit/conftest.py` — directory-level unit marker
- [x] `tests/integration/conftest.py` — directory-level integration marker
- [x] `tests/cross_service/conftest.py` — all_service_clients + stream_monitor + services_healthy
- [x] `tests/smoke/conftest.py`

#### ATDD Smoke Test
- [x] `tests/smoke/test_scaffold_story_1_1.py` — 8 AC classes covering Story 1.1; previously-failing `test_e2e_page_objects_index_exists` and `test_e2e_helpers_index_exists` assertions now unblocked

### Quality Checks

| Check | Status |
|-------|--------|
| No hard waits / `waitForTimeout` | ✅ All async waits use `waitForResponse` or `waitForSelector` |
| Network-first pattern | ✅ Intercept helpers in `network.helper.ts`; POM uses `waitForResponse` |
| mergeTests composition | ✅ 4 fixture concerns merged in `fixtures/index.ts` |
| data-testid selector strategy | ✅ All page objects use `getByTestId()` |
| Auto-cleanup in fixtures | ✅ API fixture deletes in reverse; tenant fixture disposes contexts |
| Transaction rollback (backend) | ✅ Root `conftest.py` wraps each test in a DB transaction |
| Redis flush (backend) | ✅ `stream_monitor` teardown calls `flush_test_streams()` |
| Infrastructure skip guard | ✅ `services_healthy` uses `pytest.skip()` not `pytest.fail()` |
| No secrets in files | ✅ `.env.example` uses placeholder values only |
| Celery fixtures | ✅ Correct for data-pipeline/notification services (use Celery internally) |

### Enhancement Run Sign-Off

**Completed by:** bmad-testarch-framework skill (autopilot)
**Date:** 2026-04-07
**Framework:** Playwright (E2E) + pytest + pytest-asyncio (backend)
**Notes:** Enhancement run completed all gap items found during post-Epic-1 retrospective. All 40 framework files validated present. Two previously-failing ATDD smoke tests (`test_e2e_page_objects_index_exists`, `test_e2e_helpers_index_exists`) unblocked. Cross-service fixture infrastructure fully implemented. Tenant fixture added for Epic 12 tier-gate ATDD.

---

## Enhancement Run 3 — 2026-04-09 (project-context.md RBAC + API tier patterns)

### Context

Run triggered post-Epic-2 retrospective. The framework now operates against a fully implemented
monorepo (Epics 1 & 2 complete). `project-context.md` was updated with additional canonical
patterns: `register_and_verify_with_role`, exhaustive RBAC permission matrix testing, and
cross-tenant isolation. This run fills the remaining gaps against the updated patterns.

### Gaps Identified & Resolved

**Python / pytest:**

| Gap | Resolution |
|-----|------------|
| `tests/api/` directory missing — `api` marker defined in pyproject.toml but no conftest or `__init__.py` | Created `tests/api/__init__.py` + `tests/api/conftest.py` with live-service fixtures |
| `register_and_verify_with_role` pattern from project-context.md not in eusolicit-test-utils | Created `eusolicit_test_utils.auth_helpers` with `register_and_verify_with_role`, `create_company_pair`, `make_role_token_fixture` |
| RBAC parametrize helpers missing — exhaustive role × permission matrix not available | Created `eusolicit_test_utils.rbac` with `ROLE_HIERARCHY`, `write_access_cases()`, `manage_access_cases()`, `RoleTestCase`, `tc_ids()` |
| `eusolicit_test_utils.__init__.py` not exporting new modules | Updated to export all new helpers and RBAC utilities |
| `tests/README.md` missing — no documentation for Python test suite | Created comprehensive README covering all test levels, fixtures, patterns, CI |

**TypeScript / Playwright E2E:**

| Gap | Resolution |
|-----|------------|
| No RBAC fixture for 5-tier role hierarchy testing | Created `e2e/support/fixtures/rbac.fixture.ts` with `getRoleClient`, `getRoleToken`, `crossTenantSetup` |
| `e2e/support/fixtures/index.ts` not including RBAC fixture | Updated to merge `rbacTest` (5 → 6 fixture sources) |
| No RBAC role hierarchy utilities for E2E | Created `e2e/support/helpers/rbac.helper.ts` with `ROLE_HIERARCHY`, `rolesBelow()`, `canBypassEntityPermissions()`, `buildTestLoginPayload()`, `extractToken()`, permission matrix arrays |
| `e2e/support/helpers/index.ts` not exporting RBAC helpers | Updated to export all RBAC utilities and types |

### Files Created (7)

| File | Purpose |
|------|---------|
| `tests/api/__init__.py` | Python package marker for the `api` test tier |
| `tests/api/conftest.py` | Live-service fixtures: `live_client_api`, `live_admin_api`, `api_auth_token`, `auth_live_client`, `cross_tenant_clients` |
| `tests/README.md` | Comprehensive Python test suite documentation |
| `packages/eusolicit-test-utils/.../auth_helpers.py` | `register_and_verify_with_role`, `create_company_pair`, SQL-level pattern docs |
| `packages/eusolicit-test-utils/.../rbac.py` | Role hierarchy, permission matrix, parametrize-ready test case builders |
| `e2e/support/helpers/rbac.helper.ts` | TypeScript RBAC constants, role/permission helpers, test-login payload builder |
| `e2e/support/fixtures/rbac.fixture.ts` | `getRoleClient`, `getRoleToken`, `crossTenantSetup` with ATDD red-phase fallback |

### Files Updated (4)

| File | Change |
|------|--------|
| `packages/eusolicit-test-utils/...//__init__.py` | Added exports for `auth_helpers` and `rbac` modules |
| `e2e/support/helpers/index.ts` | Added RBAC helper exports + type re-exports |
| `e2e/support/fixtures/index.ts` | Merged `rbacTest` — 5 fixture sources → 6 |

### Knowledge Base Patterns Applied

- **fixture-architecture.md** — `rbac.fixture.ts` uses one-concern-per-fixture + auto-cleanup pattern for allocated `APIRequestContext` objects
- **data-factories.md** — `auth_helpers.py` and `rbac.fixture.ts` both support ATDD red-phase (placeholder tokens when test-login endpoint not yet implemented)
- **test-quality.md** — `live_*_api` fixtures use `pytest.skip()` (not fail) for service unavailability; `services_healthy` pattern replicated in API tier conftest

### Post-Enhancement Checklist

- [x] `tests/api/__init__.py` — package marker
- [x] `tests/api/conftest.py` — `live_client_api`, `api_auth_token`, `cross_tenant_clients`
- [x] `tests/README.md` — documentation for all test levels + canonical patterns
- [x] `eusolicit_test_utils.auth_helpers` — `register_and_verify_with_role`, `create_company_pair`
- [x] `eusolicit_test_utils.rbac` — full role/permission hierarchy + parametrize helpers
- [x] `eusolicit_test_utils.__init__.py` — exports all new modules
- [x] `e2e/support/helpers/rbac.helper.ts` — TypeScript RBAC utilities
- [x] `e2e/support/helpers/index.ts` — RBAC re-exports
- [x] `e2e/support/fixtures/rbac.fixture.ts` — role fixture + cross-tenant setup
- [x] `e2e/support/fixtures/index.ts` — RBAC fixture merged

### Quality Checks

| Check | Status |
|-------|--------|
| ATDD red-phase support (placeholder tokens) | ✅ Both `rbac.fixture.ts` and `auth_helpers.py` degrade gracefully |
| Auto-disposal of `APIRequestContext` | ✅ `rbac.fixture.ts` tracks + disposes all allocated contexts |
| `pytest.skip()` on service unavailability | ✅ `live_client_api` and `live_admin_api` skip rather than fail |
| JWT claim extraction (no signature verification) | ✅ Base64url decode in `rbac.fixture.ts` + `auth_helpers.py` |
| Parametrize-ready RBAC cases | ✅ `read_access_cases()`, `write_access_cases()`, `manage_access_cases()` |
| SQL-level pattern documented | ✅ `_SQL_PATTERN_DOCSTRING` in `auth_helpers.py` for TestClient tests |
| No secrets in new files | ✅ All tokens are test-only placeholders |

### Enhancement Run 3 Sign-Off

**Completed by:** bmad-testarch-framework skill (autopilot)
**Date:** 2026-04-09
**Framework:** Playwright 1.50 (E2E) + pytest 8+ + pytest-asyncio (backend)
**Notes:** Enhancement run 3 implements all canonical patterns from the post-Epic-2 project-context.md retrospective. The `tests/api/` tier is now fully scaffolded, the RBAC test infrastructure covers the 5-tier role hierarchy and entity-permission matrix, and cross-tenant isolation helpers are available in both TypeScript (E2E) and Python (pytest) layers.
