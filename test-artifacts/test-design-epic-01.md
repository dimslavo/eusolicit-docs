---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-06'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 1
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-01-infrastructure-foundation.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
---

# Test Design: Epic 1 — Infrastructure & Monorepo Foundation

**Date:** 2026-04-06
**Author:** TEA Master Test Architect
**Status:** Draft

---

## Executive Summary

**Scope:** Epic-level test design for E01 — Infrastructure & Monorepo Foundation. Covers monorepo scaffold, Docker Compose local dev environment, PostgreSQL schema design with role-based access, Alembic migrations, Redis Streams event bus, 3 shared packages (eusolicit-common, eusolicit-models, eusolicit-kraftdata), GitHub Actions CI pipeline, Helm chart templates, and Terraform scaffold. 10 stories, 34 points, Sprints 1–2.

**Risk Summary:**

- Total risks identified: 11
- High-priority risks (≥6): 2 (schema isolation enforcement, Redis Streams dead letter handling)
- Critical categories: SEC (schema isolation), DATA (event delivery), TECH (shared package contracts), OPS (environment stability)

**Coverage Summary:**

- P0 scenarios: 5 (~8–15 hours)
- P1 scenarios: 11 (~12–20 hours)
- P2 scenarios: 11 (~6–12 hours)
- P3 scenarios: 4 (~2–4 hours)
- **Total effort**: ~28–51 hours (~1–1.5 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **Feature-level business logic** | E01 is infrastructure only; no user-facing features | Covered in E02+ test designs |
| **KraftData AI integration testing** | No AI features in E01; kraftdata package is types-only | Type validation tested; HTTP client deferred to E03+ |
| **Stripe billing integration** | No billing in E01 | Covered in billing epic test design |
| **Auth provider integration** | Auth middleware is abstract in E01 (no JWT provider) | Concrete auth testing deferred to auth epic |
| **Load/performance testing** | Infrastructure not under production load yet | k6 tests deferred to E10+ when services have real endpoints |
| **Terraform resource provisioning** | E01 creates scaffold only; no actual cloud resources | `terraform validate` confirms syntax; provisioning tested when modules are implemented |
| **Frontend feature testing** | E01 frontend is a Next.js placeholder | Verified dev server starts; feature tests in E09+ |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E01-R-001** | **SEC** | PostgreSQL schema isolation enforcement failure — misconfigured DB roles allow cross-schema writes, violating data boundaries that all future epics depend on | 2 | 3 | **6** | Automated negative access tests per role; verify `ALTER DEFAULT PRIVILEGES` applied; CI regression test | Backend Lead | Sprint 1 |
| **E01-R-002** | **DATA** | Redis Streams dead letter handling gaps — failed events silently lost when consumer fails 3+ times; no DLQ stream verification | 2 | 3 | **6** | Integration test: force consumer failure → verify event in `<stream>.dlq`; monitor DLQ stream length | Backend Lead | Sprint 2 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E01-R-003 | OPS | Docker Compose environment instability — 9+ containers with health check dependencies may fail non-deterministically on cold start | 2 | 2 | 4 | Health check start periods tuned (especially ClamAV ~30s); CI retry logic; `docker compose ps` assertion | DevOps |
| E01-R-004 | DATA | Alembic migration ordering conflict — 5 services running migrations against shared PostgreSQL may conflict if run out of order | 2 | 2 | 4 | `make migrate-all` enforces dependency order; CI runs migrations sequentially; each service uses own `alembic_version` table in its schema | Backend Lead |
| E01-R-005 | TECH | Shared package editable install breakage — relative path references in pyproject.toml may fail in Docker builds or CI | 2 | 2 | 4 | Test `pip install -e .` in Docker and CI; verify all 3 packages importable from each service | Backend Lead |
| E01-R-006 | OPS | CI pipeline matrix build timeout — 8-job matrix with pip install + mypy may exceed 10-minute target | 2 | 2 | 4 | Pip cache keyed by pyproject.toml hash; parallelized matrix; monitor build times | DevOps |
| E01-R-007 | TECH | Redis Streams event serialization inconsistency — JSON envelope schema drift between EventPublisher and EventConsumer | 2 | 2 | 4 | Pydantic-validated event envelope; integration test verifying round-trip publish/consume with full envelope fields | Backend Lead |
| E01-R-008 | TECH | eusolicit-common middleware contract misalignment — abstract base classes may not match concrete implementation needs in later epics | 2 | 2 | 4 | Unit tests for abstract interface contracts; review against PRD feature requirements | Backend Lead |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E01-R-009 | OPS | Helm chart template rendering errors not caught until deployment | 1 | 2 | 2 | `helm template` validation in CI |
| E01-R-010 | OPS | ClamAV startup delay causes false unhealthy status in Docker Compose | 2 | 1 | 2 | Generous health check start period (45s) |
| E01-R-011 | OPS | Terraform scaffold diverges from actual infrastructure needs | 1 | 1 | 1 | Monitor; update when modules are implemented |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

### System-Level Risk Inheritance

- E01-R-001 (schema isolation) → extends system R-002 (multi-tenant data isolation). E01 establishes the DB role foundation that R-002 depends on.
- E01-R-002 (Redis DLQ) → extends system R-007 (Redis Streams delivery). E01 implements the dead letter handling that R-007 identified as needed.

---

## Entry Criteria

- [ ] Epic 1 stories reviewed and accepted by team
- [ ] Docker, Docker Compose, Python 3.12, Node.js available on dev machines
- [ ] GitHub repository created with branch protection rules
- [ ] PostgreSQL 16, Redis 7, MinIO, ClamAV Docker images available
- [ ] Helm 3 and Terraform >= 1.5 installed for chart/module validation

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95% or failures triaged)
- [ ] No open high-priority/high-severity bugs
- [ ] `docker compose up` starts all services with health checks green
- [ ] CI pipeline green on merge to main
- [ ] DB role isolation verified with negative access tests
- [ ] Redis Streams event bus smoke test passing

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on if time-constrained), NOT execution timing. See "Execution Strategy" for when tests run.

### P0 (Critical)

**Criteria:** Blocks core infrastructure + High risk (≥6) + No workaround + All future epics depend on it

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| PostgreSQL 6 schemas created + role access boundaries enforced | Integration | E01-R-001 | 1 | QA | Verify each of 6 roles: own schema CRUD + shared SELECT + cross-schema write rejected |
| Docker Compose full stack healthy | Integration | E01-R-003 | 1 | QA | `docker compose up` → all 9+ containers report healthy via `docker compose ps` |
| Redis Streams publish/consume smoke test | Integration | E01-R-002 | 1 | QA | Publish JSON event to `opportunities` stream → consume from `pipeline-workers` → assert payload match + envelope fields |
| All 5 services import all 3 shared packages | Unit | E01-R-005 | 1 | QA | `pip install -e .` succeeds for each service; import eusolicit_common, eusolicit_models, eusolicit_kraftdata |
| CI pipeline lint + type check + test green for all 8 projects | Integration | E01-R-006 | 1 | QA | Push to main → matrix build completes; ruff, mypy, pytest pass for all services and packages |

**Total P0:** 5 tests, ~8–15 hours

### P1 (High)

**Criteria:** Important infrastructure + Medium risk (3–5) + Foundation for feature development

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| PostgreSQL negative access — client_api_role cannot write to admin schema | Integration | E01-R-001 | 1 | QA | S01.03 AC: INSERT to `admin` schema as `client_api_role` → error |
| Alembic `upgrade head` for all 5 services | Integration | E01-R-004 | 1 | QA | `make migrate-all` → all 5 services migrate successfully; `alembic_version` in each service's schema |
| Redis Streams dead letter handling | Integration | E01-R-002 | 1 | QA | Force consumer failure 3 times → event appears in `<stream>.dlq` stream |
| Redis Streams 7 streams + 4 consumer groups exist | Integration | E01-R-007 | 1 | QA | Bootstrap script → XINFO STREAM for all 7 streams; XINFO GROUPS for all 4 groups |
| eusolicit-common config loader — env vars + .env fallback + Pydantic validation | Unit | E01-R-008 | 1 | QA | Valid config loads; missing required var raises; .env fallback works |
| eusolicit-common health endpoint — `/healthz` returns status + version + checks | Unit | E01-R-008 | 1 | QA | Response matches `HealthResponse` schema; db/redis checks present |
| eusolicit-common exception handlers — domain exceptions → HTTP error envelope | Unit | E01-R-008 | 1 | QA | Map 400/401/403/404/409/422/500; verify `{error, message, details, correlation_id}` |
| eusolicit-common Pydantic base models — serialization round-trip | Unit | — | 1 | QA | `BaseSchema` camelCase alias, `PaginatedResponse[T]`, `ErrorResponse` |
| eusolicit-models DTOs — all 6 DTOs serialize/deserialize correctly | Unit | — | 1 | QA | Round-trip for Opportunity, Agent, Subscription, Notification, Task, Tenant |
| eusolicit-models event schemas — BaseEvent inheritance + Literal discriminators | Unit | E01-R-007 | 1 | QA | All 6 event types validate; discriminator field works for pattern matching |
| eusolicit-kraftdata models — all request/response models instantiate and validate | Unit | — | 1 | QA | 5 request + 5 response models; no HTTP dependency; field validation |

**Total P1:** 11 tests, ~12–20 hours

### P2 (Medium)

**Criteria:** Secondary infrastructure + Low risk + Configuration validation

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| Docker Compose profiles — `--profile infra-only` starts only databases | Integration | — | 1 | QA | Only PostgreSQL, Redis, MinIO, ClamAV start; services excluded |
| MinIO bucket creation — default `eusolicit` bucket exists on startup | Integration | — | 1 | QA | Init container creates bucket; MinIO API accessible on :9000 |
| ClamAV accessibility — container reachable on port 3310 after startup | Integration | E01-R-010 | 1 | QA | TCP connect to :3310 after health check passes |
| Makefile shortcuts — `make up`, `make down`, `make logs`, `make reset-db` | Integration | — | 1 | QA | Each command executes without error |
| Helm chart rendering — `helm template` validates for each of 5 services | Unit | E01-R-009 | 1 | QA | No rendering errors; valid YAML output per service values file |
| eusolicit-common structured logging — JSON output + correlation_id + service_name | Unit | — | 1 | QA | Production mode: JSON; dev mode: pretty-print; context fields present |
| eusolicit-common audit middleware — logs method, path, status, duration, user_id | Unit | — | 1 | QA | Request/response logged with all required fields |
| eusolicit-common rate limit middleware — abstract interface defined | Unit | — | 1 | QA | Interface exists; no-op backend instantiates without error |
| eusolicit-common auth middleware — abstract JWT extraction populates request state | Unit | — | 1 | QA | Abstract class instantiable with mock implementation |
| Redis Streams idempotent bootstrap — re-running creation script doesn't fail | Integration | — | 1 | QA | Run bootstrap twice → no errors; streams/groups unchanged |
| CI pip dependency caching — cache hit on repeat run | Integration | E01-R-006 | 1 | QA | Second CI run shows cache restored; faster than first run |

**Total P2:** 11 tests, ~6–12 hours

### P3 (Low)

**Criteria:** Nice-to-have + Validation benchmarks

| Requirement | Test Level | Test Count | Owner | Notes |
|-------------|------------|------------|-------|-------|
| Terraform validate — `terraform validate` passes on root module | Unit | 1 | QA | Scaffold syntax valid |
| Frontend Next.js placeholder — dev server starts on `npm run dev` | Integration | 1 | QA | HTTP 200 on localhost |
| Docker image size — each service image < 200MB | Integration | 1 | QA | Multi-stage build verification |
| CI workflow timing — completes in under 10 minutes for clean run | Integration | 1 | QA | Monitor matrix build wall-clock time |

**Total P3:** 4 tests, ~2–4 hours

---

## Execution Strategy

**Philosophy:** Run everything in PRs if <15 min; defer only if expensive/long-running.

### Every PR: pytest + Integration Tests (~10–15 min)

All functional tests from P0–P3 run on every PR:
- Unit tests (shared packages, middleware, models): pytest with pytest-asyncio
- Integration tests (Docker Compose health, DB schema verification, Redis Streams, Alembic migrations): pytest against Docker Compose services
- Helm template validation: `helm template` in CI
- Terraform validation: `terraform validate` in CI
- Parallelized via pytest-xdist workers

### On Merge to Main: Full CI Matrix

GitHub Actions matrix build (8 parallel jobs):
- ruff lint + mypy type check + pytest for all 5 services and 3 packages
- Validates shared package imports and CI pipeline health

### Not Applicable for E01

- **Nightly k6 performance tests**: No production endpoints to load test yet
- **Weekly chaos tests**: No multi-service interactions requiring fault injection yet

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Effort Range | Notes |
|----------|-------|-------------|-------|
| P0 | 5 | ~8–15 hours | Docker Compose + DB schema setup complex; Redis Streams integration |
| P1 | 11 | ~12–20 hours | Shared package unit tests straightforward; migration/DLQ tests need setup |
| P2 | 11 | ~6–12 hours | Configuration validation; mostly simple assertions |
| P3 | 4 | ~2–4 hours | Benchmark/validation checks |
| **Total** | **31** | **~28–51 hours** | **~1–1.5 weeks (1 QA)** |

### Prerequisites

**Test Data:**
- PostgreSQL init script creates schemas and roles (S01.03)
- Redis bootstrap script creates streams and consumer groups (S01.05)
- No user/company factories needed for E01 (infrastructure only)

**Tooling:**
- pytest + pytest-asyncio for backend integration/unit tests
- Docker Compose for local integration environment
- psycopg2/asyncpg for direct DB role verification
- redis-py for Redis Streams assertion
- helm CLI for chart template validation
- terraform CLI for scaffold validation

**Environment:**
- Docker Compose with all 9+ containers running
- Python 3.12 with all shared packages in editable mode
- GitHub Actions runner with Python 3.12 + Docker

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate**: 100% (no exceptions)
- **P1 pass rate**: ≥95% (waivers required for failures)
- **P2/P3 pass rate**: ≥90% (informational)
- **High-risk mitigations**: 100% complete or approved waivers

### Coverage Targets

- **Infrastructure correctness**: ≥80% (schema isolation, event bus, migrations)
- **Security scenarios**: 100% (all DB role boundary tests must pass)
- **Shared package unit tests**: ≥80% coverage on eusolicit-common (per S01.06 AC)

### Non-Negotiable Requirements

- [ ] All P0 tests pass
- [ ] No high-risk (≥6) items unmitigated
- [ ] Schema isolation tests (SEC category) pass 100%
- [ ] `docker compose up` produces all-healthy state

---

## Mitigation Plans

### E01-R-001: PostgreSQL Schema Isolation Enforcement Failure (Score: 6)

**Mitigation Strategy:**
1. Init SQL script uses `ALTER DEFAULT PRIVILEGES` so future tables inherit correct permissions automatically
2. Automated negative access test: for each of 6 service roles, attempt INSERT/UPDATE/DELETE on every non-owned schema → verify permission denied
3. CI regression test prevents drift when new migrations add tables
4. `shared.tenants` table verified as SELECT-only for non-migration roles

**Owner:** Backend Lead
**Timeline:** Sprint 1
**Status:** Planned
**Verification:** P0 integration test + P1 negative access test

### E01-R-002: Redis Streams Dead Letter Handling Gaps (Score: 6)

**Mitigation Strategy:**
1. EventConsumer tracks per-message failure count via XPENDING
2. After 3 failures, message XACK'd from original stream and XADD'd to `<stream>.dlq`
3. Integration test: publish event → force consumer exception 3 times → assert event in DLQ stream with original payload
4. Monitoring alert on DLQ stream length > 0

**Owner:** Backend Lead
**Timeline:** Sprint 2
**Status:** Planned
**Verification:** P1 integration test for DLQ flow

---

## Assumptions and Dependencies

### Assumptions

1. Docker Compose v2 available on all developer machines and CI runners
2. PostgreSQL 16 Docker image supports `IF NOT EXISTS` / `DO $$ ... $$` for idempotent init scripts
3. Redis 7 Docker image supports Streams with `MKSTREAM` option
4. GitHub Actions runners have sufficient resources for 8-job parallel matrix
5. Python 3.12 is the minimum version; `src/` layout used for all services
6. Helm 3 and Terraform >= 1.5 are available in CI for validation steps

### Dependencies

1. **Docker images** — PostgreSQL 16, Redis 7, MinIO, ClamAV official images available — Required by Sprint 1
2. **GitHub Actions** — Repository with branch protection and Actions enabled — Required by Sprint 1
3. **Python 3.12** — Available on CI runners — Required by Sprint 1

### Risks to Plan

- **Risk**: ClamAV signature database update during Docker build slows cold start
  - **Impact**: Docker Compose health checks timeout on first start
  - **Contingency**: Pre-built ClamAV image with cached signatures; generous health check start period (45s+)

- **Risk**: Shared package relative path references break in non-standard environments
  - **Impact**: Services fail to import shared packages in Docker or CI
  - **Contingency**: Test `pip install -e .` in both local and Docker contexts; fallback to absolute path references if needed

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
|-------------------|--------|-----------------|
| **PostgreSQL schemas + roles** | Foundation for all service data access; E02+ depend on schema boundaries | P0 schema isolation test; P1 negative access test |
| **Redis Streams event bus** | Foundation for all async inter-service communication; E02+ depend on EventPublisher/EventConsumer | P0 smoke test; P1 DLQ test; P1 streams/groups verification |
| **eusolicit-common** | Every service depends on config, logging, middleware, health, exceptions | P1 unit tests for all public interfaces |
| **eusolicit-models** | Inter-service contracts; breaking changes affect all consumers | P1 DTO + event schema round-trip tests |
| **eusolicit-kraftdata** | Type definitions for AI Gateway integration; E03+ depend on these types | P1 model validation tests |
| **Docker Compose** | Local dev environment for all developers; E02+ development workflow depends on it | P0 all-healthy test; P2 profile + shortcut tests |
| **CI pipeline** | Gate for all future PRs; broken CI blocks all development | P0 green pipeline test |
| **Helm charts** | Deployment template for all services; E10+ depend on correct rendering | P2 helm template validation |

---

## Follow-on Workflows (Manual)

- Run `*atdd` to generate failing P0 tests (separate workflow; not auto-run).
- Run `*automate` for broader coverage once implementation exists.

---

## Approval

**Test Design Approved By:**

- [ ] Product Manager: __________ Date: __________
- [ ] Tech Lead: __________ Date: __________
- [ ] QA Lead: __________ Date: __________

**Comments:**

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk classification framework (P × I scoring, gate decisions)
- `probability-impact.md` — Risk scoring methodology (1–3 scales)
- `test-levels-framework.md` — Test level selection (E2E vs API vs Unit)
- `test-priorities-matrix.md` — P0–P3 prioritization criteria

### Related Documents

- PRD: `eusolicit-docs/EU_Solicit_PRD_v1.md`
- Epic: `eusolicit-docs/planning-artifacts/epic-01-infrastructure-foundation.md`
- Architecture: `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md`
- System-Level Test Design (Architecture): `eusolicit-docs/test-artifacts/test-design-architecture.md`
- System-Level Test Design (QA): `eusolicit-docs/test-artifacts/test-design-qa.md`

### Existing Test Coverage (Pre-E01)

| Area | Files | Status |
|------|-------|--------|
| Backend test infrastructure | 5 service conftest.py + root conftest.py | Fixtures built; test directories empty |
| E2E test infrastructure | 7 Playwright specs (auth, opportunities, smoke, analytics, admin, enterprise) | Infrastructure tests ready; some in ATDD RED phase |
| Shared test utilities | eusolicit-test-utils package (factories, ServiceClient, DB helpers, Redis helpers, JWT generation) | Available for E01 test development |

---

**Generated by:** BMad TEA Agent — Test Architect Module
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
