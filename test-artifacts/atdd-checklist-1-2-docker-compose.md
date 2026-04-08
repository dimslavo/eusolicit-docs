---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-06'
workflowType: 'testarch-atdd'
storyId: '1-2-docker-compose-local-development-environment'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/1-2-docker-compose-local-development-environment.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-01.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
---

# ATDD Checklist: Story 1.2 — Docker Compose Local Development Environment

## Story Summary

**Epic:** E01 — Infrastructure & Monorepo Foundation
**Story:** 1.2 — Docker Compose Local Development Environment
**Sprint:** 1 | **Priority:** P0 (Docker Compose is prerequisite for all local development from Sprint 2+)
**Risk-Driven:** Yes — E01-R-003 (Docker Compose environment instability, Score: 4), E01-R-010 (ClamAV startup delay, Score: 2)

**Focus Areas:**
- docker-compose.yml with 9+ containers: 5 services + 4 infra (AC1)
- PostgreSQL 16 on localhost:5432 with seeded superuser (AC2)
- Redis 7 on localhost:6379 (AC3)
- MinIO on localhost:9000/9001 with eusolicit bucket created (AC4)
- ClamAV on localhost:3310 with generous start_period (AC5)
- Volume mounts for hot-reload development (AC6)
- Health checks on every container (AC7)
- .env.example EXTENDED with Docker Compose variables (AC8)
- Makefile EXTENDED with Docker Compose shortcuts (AC9)
- .dockerignore, minimal FastAPI main.py per service, infra/postgres/init/ mount point

---

## TDD Red Phase (Current)

**Status:** RED — All tests use `@pytest.mark.skip(reason="ATDD RED: Docker Compose environment not implemented — Story 1.2")` and will be skipped until features are implemented.

| Test Class | File | Test Functions | Expanded Tests | AC | Status |
|------------|------|---------------|----------------|----|----|
| `TestAC1DockerComposeContainers` | `tests/smoke/test_docker_compose_story_1_2.py` | 6 | 13 | AC1 | Skipped (RED) |
| `TestAC2PostgresService` | `tests/smoke/test_docker_compose_story_1_2.py` | 5 | 5 | AC2 | Skipped (RED) |
| `TestAC3RedisService` | `tests/smoke/test_docker_compose_story_1_2.py` | 3 | 3 | AC3 | Skipped (RED) |
| `TestAC4MinioService` | `tests/smoke/test_docker_compose_story_1_2.py` | 8 | 8 | AC4 | Skipped (RED) |
| `TestAC5ClamavService` | `tests/smoke/test_docker_compose_story_1_2.py` | 4 | 4 | AC5 | Skipped (RED) |
| `TestAC6VolumeHotReload` | `tests/smoke/test_docker_compose_story_1_2.py` | 5 | 25 | AC6 | Skipped (RED) |
| `TestAC7HealthChecks` | `tests/smoke/test_docker_compose_story_1_2.py` | 6 | 14 | AC7 | Skipped (RED) |
| `TestAC8EnvExampleExtended` | `tests/smoke/test_docker_compose_story_1_2.py` | 15 | 15 | AC8 | Skipped (RED) |
| `TestAC9MakefileExtended` | `tests/smoke/test_docker_compose_story_1_2.py` | 11 | 11 | AC9 | Skipped (RED) |
| `TestDockerIgnore` | `tests/smoke/test_docker_compose_story_1_2.py` | 3 | 7 | — | Skipped (RED) |
| `TestMinimalFastAPIApps` | `tests/smoke/test_docker_compose_story_1_2.py` | 4 | 20 | AC7 | Skipped (RED) |
| `TestInfraPostgresInit` | `tests/smoke/test_docker_compose_story_1_2.py` | 2 | 2 | — | Skipped (RED) |
| `TestDockerComposeVolumes` | `tests/smoke/test_docker_compose_story_1_2.py` | 4 | 4 | — | Skipped (RED) |
| **Total** | **1 file** | **76 functions** | **131 test cases** | **AC1–AC9** | **All skipped** |

---

## Acceptance Criteria Coverage

### AC1: docker compose up starts all 9+ containers

| AC Detail | Test | Priority |
|-----------|------|----------|
| docker-compose.yml exists | `test_docker_compose_file_exists` | P0 |
| docker-compose.yml is valid YAML with services key | `test_docker_compose_parses_as_valid_yaml` | P0 |
| Each infra service defined (postgres, redis, minio, clamav) | `test_infrastructure_service_defined[<svc>]` | P0 |
| minio-init init container defined | `test_minio_init_service_defined` | P0 |
| Each app service defined (client-api, admin-api, data-pipeline, ai-gateway, notification) | `test_application_service_defined[<svc>]` | P0 |
| At least 9 services total | `test_at_least_nine_services_defined` | P0 |

### AC2: PostgreSQL 16 accessible on localhost:5432 with superuser

| AC Detail | Test | Priority |
|-----------|------|----------|
| Uses postgres:16-alpine image | `test_postgres_uses_pg16_alpine_image` | P0 |
| Maps port 5432 | `test_postgres_maps_port_5432` | P0 |
| Has superuser env vars (POSTGRES_USER/POSTGRES_PASSWORD) | `test_postgres_has_superuser_env_vars` | P0 |
| Uses named volume for data persistence | `test_postgres_has_named_volume` | P0 |
| Mounts infra/postgres/init/ for Story 1.3 forward-compatibility | `test_postgres_mounts_init_directory` | P1 |

### AC3: Redis 7 accessible on localhost:6379

| AC Detail | Test | Priority |
|-----------|------|----------|
| Uses redis:7-alpine image | `test_redis_uses_redis7_alpine_image` | P0 |
| Maps port 6379 | `test_redis_maps_port_6379` | P0 |
| Uses named volume | `test_redis_has_named_volume` | P1 |

### AC4: MinIO accessible on 9000 (API) / 9001 (console) + eusolicit bucket

| AC Detail | Test | Priority |
|-----------|------|----------|
| Uses minio/minio image | `test_minio_uses_official_image` | P0 |
| Maps port 9000 (API) | `test_minio_maps_port_9000` | P0 |
| Maps port 9001 (console) | `test_minio_maps_port_9001` | P0 |
| Server command with --console-address | `test_minio_has_server_command_with_console_address` | P0 |
| Uses named volume | `test_minio_has_named_volume` | P1 |
| minio-init creates eusolicit bucket | `test_minio_init_creates_eusolicit_bucket` | P0 |
| minio-init depends on minio healthy | `test_minio_init_depends_on_minio_healthy` | P0 |
| minio-init uses minio/mc image | `test_minio_init_uses_mc_image` | P0 |

### AC5: ClamAV accessible on localhost:3310

| AC Detail | Test | Priority |
|-----------|------|----------|
| Uses clamav/clamav image | `test_clamav_uses_official_image` | P0 |
| Maps port 3310 | `test_clamav_maps_port_3310` | P0 |
| Uses named volume for signatures | `test_clamav_has_named_volume` | P1 |
| Health check start_period >= 60s (E01-R-010 mitigation) | `test_clamav_health_check_has_generous_start_period` | **P0** |

### AC6: Volume mounts for hot-reload

| AC Detail | Test (×5 services) | Priority |
|-----------|---------------------|----------|
| Each service mounts source code volume | `test_service_mounts_source_code_volume[<svc>]` | P0 |
| Each service has --reload flag | `test_service_has_reload_flag[<svc>]` | P0 |
| Each service maps correct port (8001–8005) | `test_service_maps_correct_port[<svc>-<port>]` | P0 |
| Each service depends on postgres healthy | `test_service_depends_on_postgres_healthy[<svc>]` | P0 |
| Each service depends on redis healthy | `test_service_depends_on_redis_healthy[<svc>]` | P0 |

### AC7: Health checks for every container

| AC Detail | Test | Priority |
|-----------|------|----------|
| postgres healthcheck uses pg_isready | `test_postgres_has_healthcheck` | P0 |
| redis healthcheck uses redis-cli ping | `test_redis_has_healthcheck` | P0 |
| minio healthcheck uses health/live | `test_minio_has_healthcheck` | P0 |
| clamav has healthcheck | `test_clamav_has_healthcheck` | P0 |
| Each app service healthcheck uses /healthz on correct port | `test_application_service_has_healthcheck[<svc>]` | P0 |
| Each app service healthcheck has start_period | `test_application_service_healthcheck_has_start_period[<svc>]` | P0 |

### AC8: .env.example EXTENDED (not replaced)

| AC Detail | Test | Priority |
|-----------|------|----------|
| Preserves existing TEST_ENV=local | `test_env_example_preserves_existing_test_env` | **P0** |
| Preserves existing service URLs | `test_env_example_preserves_existing_service_urls` | **P0** |
| Preserves existing TEST_DATABASE_URL | `test_env_example_preserves_existing_test_database_url` | **P0** |
| Preserves existing TEST_REDIS_URL | `test_env_example_preserves_existing_redis_url` | P0 |
| Preserves existing TEST_JWT_SECRET | `test_env_example_preserves_existing_jwt_config` | P0 |
| Adds POSTGRES_USER | `test_env_example_has_postgres_user` | P0 |
| Adds POSTGRES_PASSWORD | `test_env_example_has_postgres_password` | P0 |
| Adds POSTGRES_DB | `test_env_example_has_postgres_db` | P0 |
| Adds REDIS_URL | `test_env_example_has_redis_url` | P0 |
| Adds MINIO_ROOT_USER | `test_env_example_has_minio_root_user` | P0 |
| Adds MINIO_ROOT_PASSWORD | `test_env_example_has_minio_root_password` | P0 |
| Adds MINIO_ENDPOINT | `test_env_example_has_minio_endpoint` | P0 |
| Adds CLAMAV_HOST | `test_env_example_has_clamav_host` | P0 |
| Adds per-service DATABASE_URL vars | `test_env_example_has_per_service_db_urls` | P0 |
| Docker section uses container hostnames (not localhost) | `test_env_example_docker_section_uses_container_hostnames` | P1 |

### AC9: Makefile EXTENDED (not replaced)

| AC Detail | Test | Priority |
|-----------|------|----------|
| Preserves existing test target | `test_makefile_preserves_existing_test_target` | **P0** |
| Preserves existing lint target | `test_makefile_preserves_existing_lint_target` | **P0** |
| Preserves existing coverage target | `test_makefile_preserves_existing_coverage_target` | **P0** |
| Preserves existing quality-check target | `test_makefile_preserves_existing_quality_check_target` | **P0** |
| Adds `make up` target | `test_makefile_has_up_target` | P0 |
| Adds `make down` target | `test_makefile_has_down_target` | P0 |
| Adds `make logs` target | `test_makefile_has_logs_target` | P0 |
| Adds `make reset-db` target | `test_makefile_has_reset_db_target` | P0 |
| Adds `make ps` target | `test_makefile_has_ps_target` | P1 |
| Adds `make infra` target | `test_makefile_has_infra_target` | P1 |
| New targets declared in .PHONY | `test_makefile_up_target_in_phony` | P2 |

---

## Epic Test Design Traceability

| Epic Test Design Requirement | AC | Test Class(es) | Priority |
|------------------------------|-----|----------------|----------|
| P0: Docker Compose full stack healthy | AC1, AC7 | `TestAC1DockerComposeContainers`, `TestAC7HealthChecks` | **P0** |
| P2: Docker Compose profiles (infra-only) | AC1 | Not tested structurally — runtime validation | P2 |
| P2: MinIO bucket creation | AC4 | `TestAC4MinioService` | P0 |
| P2: ClamAV accessibility (port 3310) | AC5 | `TestAC5ClamavService` | P0 |
| P2: Makefile shortcuts | AC9 | `TestAC9MakefileExtended` | P0 |
| E01-R-003: Docker Compose instability (Score: 4) | AC7 | `TestAC7HealthChecks`, `TestAC5ClamavService` | **P0** |
| E01-R-010: ClamAV startup delay (Score: 2) | AC5 | `test_clamav_health_check_has_generous_start_period` | **P0** |

---

## Fixture Needs

_None required for GREEN phase._ These tests are pure YAML parsing and filesystem assertions — no Docker, database, Redis, or running services needed.

| Resource | Purpose | Status |
|----------|---------|--------|
| Python 3.12+ runtime | YAML parsing, file system checks | Available |
| PyYAML | Parse docker-compose.yml | Available (installed) |
| File system access | Verify directory/file existence, read content | Available |
| `re` (stdlib) | Regex matching for Makefile targets | Available |

---

## Mock Requirements

_None._ Story 1.2 ATDD tests are structural validation of configuration files — no services, databases, or external systems involved.

---

## Required data-testid Attributes

_Not applicable — all tests are YAML parsing, filesystem, and text content verification. No UI._

---

## Implementation Guidance

### What Must Be Created (to Turn Tests GREEN)

**docker-compose.yml (AC1, AC2, AC3, AC4, AC5, AC6, AC7):**
- Define all infrastructure services: postgres, redis, minio, minio-init, clamav
- Define all application services: client-api, admin-api, data-pipeline, ai-gateway, notification
- Each infra service: correct image, port mapping, healthcheck, named volume
- Each app service: build context, port mapping, volume mount for hot-reload, `--reload` flag, depends_on postgres/redis healthy, healthcheck on /healthz
- minio-init: depends_on minio healthy, creates eusolicit bucket
- ClamAV: start_period >= 60s on healthcheck
- Top-level volumes: pgdata, redisdata, miniodata, clamdata

**.env.example Extension (AC8):**
- APPEND Docker Compose section below existing content
- DO NOT modify or remove existing variables (TEST_ENV, service URLs, DB URLs, Redis, JWT, Playwright)
- New section: POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB, REDIS_URL, MINIO_ROOT_USER, MINIO_ROOT_PASSWORD, MINIO_ENDPOINT, CLAMAV_HOST
- Per-service DB URLs using container hostname `postgres` (not localhost)

**Makefile Extension (AC9):**
- APPEND Docker Compose targets below existing content
- DO NOT modify or remove existing targets (test, lint, coverage, quality-check, etc.)
- New targets: up, down, logs, ps, infra, reset-db
- Declare new targets in .PHONY

**.dockerignore (deferred from Story 1.1 review):**
- Create at monorepo root
- Exclude: .venv, node_modules, .git, __pycache__, .pytest_cache, .env

**Minimal FastAPI main.py (AC7 health check support):**
- Create `src/<module>/main.py` for each of 5 services
- Each must: `from fastapi import FastAPI`, create `app = FastAPI(...)`, define `GET /healthz` returning `{"status": "ok"}`

**infra/postgres/init/ (forward-compatibility for Story 1.3):**
- Create `infra/postgres/init/` directory
- Add `.gitkeep` to track empty directory in git
- docker-compose.yml mounts this to `/docker-entrypoint-initdb.d/`

### Architecture-Mandated Port Assignments

| Service | Port | Module Path |
|---------|------|-------------|
| client-api | 8001 | `client_api.main:app` |
| admin-api | 8002 | `admin_api.main:app` |
| data-pipeline | 8003 | `data_pipeline.main:app` |
| ai-gateway | 8004 | `ai_gateway.main:app` |
| notification | 8005 | `notification.main:app` |

### Infrastructure Container Specs

| Container | Image | Ports | Health Check | Volume |
|-----------|-------|-------|-------------|--------|
| PostgreSQL | `postgres:16-alpine` | 5432 | `pg_isready` | pgdata |
| Redis | `redis:7-alpine` | 6379 | `redis-cli ping` | redisdata |
| MinIO | `minio/minio:latest` | 9000, 9001 | `/minio/health/live` | miniodata |
| ClamAV | `clamav/clamav:latest` | 3310 | TCP/ping (start_period: 60s) | clamdata |
| minio-init | `minio/mc:latest` | — | — (one-shot) | — |

---

## Red-Green-Refactor Workflow

### RED Phase (Complete — TEA)

- [x] 131 acceptance test cases generated (76 test functions, many parametrized)
- [x] All tests use `@pytest.mark.skip` to document intentional RED phase
- [x] Tests assert EXPECTED behavior against unimplemented Docker Compose config
- [x] Tests organized by acceptance criterion (AC1–AC9) in dedicated classes
- [x] Additional classes for .dockerignore, FastAPI apps, infra mount, named volumes
- [x] P0 tests for E01-R-003 (health checks) and E01-R-010 (ClamAV start_period)
- [x] Preservation tests verify .env.example and Makefile are EXTENDED, not replaced
- [x] Test file: `eusolicit-app/tests/smoke/test_docker_compose_story_1_2.py`

### GREEN Phase (Next — DEV Team)

After implementing Story 1.2 artifacts:

1. Remove `@pytest.mark.skip` from test classes one AC at a time
2. Run tests for each AC group:
   ```bash
   # Run all Story 1.2 tests
   cd eusolicit-app
   python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v

   # Run specific AC group
   python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC1"
   python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC8"

   # Run preservation tests only (critical — detect overwrite damage)
   python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "preserves"

   # Run health check tests (risk E01-R-003)
   python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC7"

   # Run ClamAV start_period test (risk E01-R-010)
   python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "clamav_health_check_has_generous"
   ```
3. Verify tests PASS (green phase)
4. If any tests fail:
   - **Implementation bug**: Fix the docker-compose.yml / .env.example / Makefile / main.py
   - **Test bug**: Fix the test assertion (update this checklist)
5. Commit passing tests

### REFACTOR Phase (DEV Team)

- Consider extracting YAML loading into a shared fixture if more compose tests are needed
- Add runtime integration tests (docker compose up → verify health) as P0 CI integration test
- Consider adding `docker compose config` validation test for compose schema correctness

---

## Execution Commands

```bash
# Navigate to app root
cd eusolicit-app

# Run ALL Story 1.2 tests (currently all skipped)
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v

# Run by AC group
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC1"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC2"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC3"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC4"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC5"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC6"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC7"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC8"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestAC9"

# Run additional structural tests
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestDockerIgnore"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestMinimalFastAPIApps"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestInfraPostgresInit"
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "TestDockerComposeVolumes"

# Run preservation tests (critical — detect overwrite damage)
python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v -k "preserves"

# Run smoke marker (includes all smoke tests from all stories)
python -m pytest -m smoke -v
```

---

## Completion Summary

| Metric | Value |
|--------|-------|
| **Story ID** | 1-2-docker-compose-local-development-environment |
| **Primary test level** | Unit / Smoke (YAML parsing + filesystem assertions) |
| **Test functions** | 76 |
| **Expanded test cases** | 131 (via parametrization across 5 services, 4 infra, 5 patterns) |
| **Test files** | 1 (`tests/smoke/test_docker_compose_story_1_2.py`) |
| **Fixtures needed** | 0 (pure YAML/filesystem/text) |
| **Mock requirements** | 0 |
| **data-testid requirements** | 0 |
| **External dependencies** | PyYAML (for docker-compose.yml parsing) |
| **Execution mode** | Parallel-safe (no shared state) |
| **TDD phase** | RED (all tests skipped) |

### Priority Breakdown

| Priority | Test Cases | AC Coverage |
|----------|------------|-------------|
| **P0** | 111 | AC1, AC2, AC3, AC4, AC5, AC6, AC7, AC8, AC9 |
| **P1** | 12 | AC2 (init mount), AC3 (volume), AC4 (volume), AC5 (volume), AC8 (hostnames), AC9 (ps, infra) |
| **P2** | 8 | AC9 (.PHONY), DockerIgnore, InfraPostgresInit, named volumes |
| **Total** | **131** | **AC1–AC9 (100%)** |

### Risk Mitigation Coverage

| Risk ID | Score | Mitigation | Test Coverage |
|---------|-------|------------|---------------|
| E01-R-003 | 4 | Health check start periods tuned; all containers have health checks | AC7 (14 tests) — all infra + app health checks validated |
| E01-R-010 | 2 | ClamAV start_period >= 60s | AC5 `test_clamav_health_check_has_generous_start_period` — verifies >= 60s |

### Preservation Safeguard Coverage

These tests specifically guard against accidental file replacement (a critical risk when Story 1.2 says EXTEND, not replace):

| File | Preservation Tests | What's Protected |
|------|-------------------|-----------------|
| `.env.example` | 5 tests | TEST_ENV, service URLs, TEST_DATABASE_URL, TEST_REDIS_URL, TEST_JWT_SECRET |
| `Makefile` | 4 tests | test, lint, coverage, quality-check targets |

### Knowledge Base References Applied

- `test-levels-framework.md` — Smoke-level test selection for structural/config verification
- `test-priorities-matrix.md` — P0 criteria (blocks all local development from Sprint 2+)
- `test-quality.md` — Deterministic assertions, parallel-safe, minimal external deps
- `risk-governance.md` — E01-R-003 and E01-R-010 mitigation verification

### Next Steps

1. **Implement Story 1.2** — Create docker-compose.yml, extend .env.example + Makefile, create main.py files
2. **Remove `@pytest.mark.skip`** — One AC group at a time, verify GREEN phase
3. **Run full test suite** — `python -m pytest tests/smoke/test_docker_compose_story_1_2.py -v`
4. **Run preservation tests first** — `pytest -k "preserves"` to catch accidental overwrites early
5. **Verify CI integration** — Tests should run as part of `make test-smoke`
6. **Proceed to Story 1.3** — PostgreSQL schema design (init SQL goes into infra/postgres/init/)

---

**Generated by:** BMad TEA Agent — ATDD Module
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
