# E01: Infrastructure & Monorepo Foundation

**Sprint**: 1--2 | **Points**: 34 | **Dependencies**: None | **Milestone**: Demo

## Goal

Establish the complete monorepo structure, local development environment, database foundation, event bus, CI pipeline, and all shared packages so that service teams can begin feature development in Sprint 3 with a fully operational, tested, and documented platform skeleton. Every service should be bootable locally via a single `docker compose up`, with migrations applied, Redis Streams flowing, and CI green on merge to main.

## Acceptance Criteria

- [ ] Monorepo root contains all 5 service directories, 3 shared packages, frontend directory, and infra directory with consistent Python project structure
- [ ] `docker compose up` starts all services, PostgreSQL 16, Redis 7, MinIO, and ClamAV with health checks passing
- [ ] PostgreSQL has 6 schemas (client, pipeline, admin, notification, gateway, shared) with per-service DB roles enforcing access boundaries
- [ ] Each service has its own Alembic migration directory with an initial empty migration that creates its schema
- [ ] Redis Streams event bus is configured with 7 streams and 4 consumer groups, and a smoke test proves publish/consume works
- [ ] GitHub Actions CI runs ruff lint, mypy type check, and pytest for all 5 services in a matrix build
- [ ] eusolicit-common provides config loader, structlog logging, base middleware, health endpoint, exception handlers, and Pydantic base models
- [ ] eusolicit-models provides shared DTOs for inter-service REST contracts and Redis Streams event schemas
- [ ] eusolicit-kraftdata provides KraftData API type definitions (request/response models only, no HTTP client)
- [ ] Helm base chart template exists with per-service values files
- [ ] Terraform scaffold contains placeholder modules for cloud resources

## Stories

### S01.01: Monorepo Scaffold & Project Structure

**Points**: 3 | **Type**: backend
**Description**: Create the full monorepo directory layout with Python project configuration for all services and packages. Each service gets a standardized structure with `src/`, `tests/`, `pyproject.toml`, and `Dockerfile`. Shared packages are installable in editable mode. The frontend directory gets a Next.js placeholder. The infra directory holds Helm and Terraform subdirectories.

**Acceptance Criteria**:
- [ ] Root `pyproject.toml` (or workspace config) references all 5 services and 3 shared packages
- [ ] Each service directory contains: `src/<service_name>/`, `tests/`, `pyproject.toml`, `Dockerfile`, `alembic.ini`
- [ ] Services: `client-api`, `admin-api`, `data-pipeline`, `ai-gateway`, `notification`
- [ ] Shared packages: `packages/eusolicit-common`, `packages/eusolicit-kraftdata`, `packages/eusolicit-models`
- [ ] `frontend/` directory contains a bootstrapped Next.js project (can be minimal placeholder)
- [ ] `infra/helm/` and `infra/terraform/` directories exist with README placeholders
- [ ] All services can import all three shared packages via editable installs
- [ ] Running `pip install -e .` in any service directory succeeds

**Implementation Notes**: Use a flat monorepo layout (no nested workspaces). Each service's `pyproject.toml` should declare dependencies on the shared packages using relative path references. Pin Python >= 3.12. Use `src/` layout to avoid import ambiguity.

---

### S01.02: Docker Compose Local Development Environment

**Points**: 5 | **Type**: devops
**Description**: Create a `docker-compose.yml` that brings up the full local development stack: all 5 Python services (with hot-reload via volume mounts), PostgreSQL 16, Redis 7, MinIO (S3-compatible), and ClamAV. Each service should have a health check. Include a `.env.example` with all required environment variables.

**Acceptance Criteria**:
- [ ] `docker compose up` starts all 9+ containers (5 services + 4 infrastructure)
- [ ] PostgreSQL 16 container is accessible on `localhost:5432` with a seeded superuser
- [ ] Redis 7 container is accessible on `localhost:6379`
- [ ] MinIO container is accessible on `localhost:9000` (API) and `localhost:9001` (console) with a default bucket created
- [ ] ClamAV container is accessible on `localhost:3310`
- [ ] All services mount source code as volumes for hot-reload during development
- [ ] Health checks are defined for every container; `docker compose ps` shows all healthy after startup
- [ ] `.env.example` documents every required environment variable with sensible local defaults
- [ ] A `Makefile` (or `justfile`) provides shortcuts: `make up`, `make down`, `make logs`, `make reset-db`

**Implementation Notes**: Use Docker Compose profiles so developers can bring up subsets (e.g., `--profile infra-only` for just databases). ClamAV takes ~30s to load signatures; set a generous health check start period. MinIO init container should create the default `eusolicit` bucket.

---

### S01.03: PostgreSQL Schema Design & Role-Based Access

**Points**: 3 | **Type**: backend
**Description**: Create an initialization SQL script that sets up 6 PostgreSQL schemas and per-service database roles with strict access boundaries. Each service role can only access its own schema plus the `shared` schema (read-only on shared for most services). Include a `shared.tenants` table as the foundational multi-tenancy reference.

**Acceptance Criteria**:
- [ ] 6 schemas created: `client`, `pipeline`, `admin`, `notification`, `gateway`, `shared`
- [ ] 6 service roles created: `client_api_role`, `admin_api_role`, `data_pipeline_role`, `ai_gateway_role`, `notification_role`, `migration_role`
- [ ] Each service role has full CRUD on its own schema and SELECT on `shared` schema
- [ ] `migration_role` has DDL privileges on all schemas (used by Alembic)
- [ ] `shared.tenants` table exists with columns: `id UUID PK`, `name`, `slug UNIQUE`, `created_at`, `updated_at`
- [ ] A test script verifies that `client_api_role` cannot write to `admin` schema (negative access test)
- [ ] Init script runs automatically on first `docker compose up` via PostgreSQL's `/docker-entrypoint-initdb.d/`

**Implementation Notes**: Use PostgreSQL's `GRANT` / `REVOKE` with `ALTER DEFAULT PRIVILEGES` so future tables in each schema automatically inherit the correct permissions. The init script should be idempotent (use `IF NOT EXISTS` / `DO $$ ... $$`).

---

### S01.04: Alembic Migration Scaffold

**Points**: 2 | **Type**: backend
**Description**: Configure Alembic for each of the 5 services with isolated migration directories. Each service targets its own schema using `include_schemas` in `env.py`. Create an initial migration per service that verifies connectivity and schema existence.

**Acceptance Criteria**:
- [ ] Each service has `alembic.ini` and `alembic/` directory under its root
- [ ] Each service's `env.py` is configured to use `search_path` set to its own schema + `shared`
- [ ] `alembic upgrade head` runs successfully for each service against the Docker Compose PostgreSQL
- [ ] Each service has one initial migration (e.g., `001_initial.py`) that is a no-op or creates a `_migrations_meta` table in its schema
- [ ] Migration naming convention: `NNN_descriptive_name.py` (sequential, not timestamp-based)
- [ ] A `make migrate-all` command runs all 5 services' migrations in dependency order

**Implementation Notes**: Set `version_table_schema` in each service's `env.py` so Alembic's `alembic_version` table lives inside the service's own schema, avoiding collisions. The `migration_role` should be used for all Alembic operations.

---

### S01.05: Redis Streams Event Bus Setup

**Points**: 3 | **Type**: backend
**Description**: Implement the Redis Streams event bus configuration and a lightweight publisher/consumer abstraction in `eusolicit-common`. Define 7 streams and 4 consumer groups. Include a health check that verifies stream existence and a smoke test that publishes and consumes a message.

**Acceptance Criteria**:
- [ ] 7 Redis Streams created on startup: `opportunities`, `agents`, `subscriptions`, `notifications`, `tasks`, `billing`, `admin`
- [ ] 4 consumer groups created: `pipeline-workers`, `notification-workers`, `ai-workers`, `admin-workers`
- [ ] `eusolicit-common` provides `EventPublisher` class with `publish(stream, event)` method
- [ ] `eusolicit-common` provides `EventConsumer` class with `consume(stream, group, consumer_name)` method
- [ ] Events are serialized as JSON with envelope: `{event_type, payload, timestamp, correlation_id, source_service}`
- [ ] A Redis init script (or Python bootstrap) creates streams and groups idempotently on service startup
- [ ] Integration test: publish an event to `opportunities` stream, consume it from `pipeline-workers` group, assert payload matches
- [ ] Dead letter handling: messages that fail 3 times are moved to a `<stream>.dlq` stream

**Implementation Notes**: Use `XADD` / `XREADGROUP` / `XACK` directly via `redis-py` (no Celery or other task framework). Consumer groups should use `MKSTREAM` option. The `EventConsumer` should support both blocking and non-blocking reads. Correlation ID enables distributed tracing.

---

### S01.06: eusolicit-common Shared Package

**Points**: 5 | **Type**: backend
**Description**: Build the foundational shared package that every service depends on. This includes configuration management, structured logging, base middleware classes, a standardized health check endpoint, exception handlers, and Pydantic base models.

**Acceptance Criteria**:
- [ ] **Config loader**: reads from environment variables with `.env` fallback, validates with Pydantic `BaseSettings`, supports per-service overrides
- [ ] **Logging**: `structlog` configured for JSON output in production, pretty-print in development; includes `correlation_id`, `service_name`, `tenant_id` in every log line
- [ ] **Auth middleware base**: abstract class that extracts and validates JWT from `Authorization` header, populates request state with user context
- [ ] **Audit middleware**: logs every request/response with method, path, status code, duration, user ID, and tenant ID
- [ ] **Rate limit middleware base**: abstract class with pluggable backend (Redis-based), configurable per-route limits
- [ ] **Health check endpoint**: `GET /healthz` returns `{status, version, uptime, checks: {db, redis, ...}}` with dependency checks
- [ ] **Exception handlers**: maps domain exceptions to HTTP responses (400/401/403/404/409/422/500) with consistent error envelope `{error, message, details, correlation_id}`
- [ ] **Pydantic base models**: `BaseSchema` (with `model_config` for camelCase alias generation), `PaginatedResponse[T]`, `ErrorResponse`, `HealthResponse`
- [ ] Unit tests for config loader, exception handlers, and Pydantic models (>= 80% coverage on this package)

**Implementation Notes**: The auth middleware should be abstract -- concrete JWT validation will be implemented later when the auth provider is integrated. Rate limit middleware should define the interface but can use a no-op backend initially. All middleware should be FastAPI-compatible (`BaseHTTPMiddleware` or Starlette middleware pattern).

---

### S01.07: eusolicit-models & eusolicit-kraftdata Shared Packages

**Points**: 5 | **Type**: backend
**Description**: Build the two remaining shared packages. `eusolicit-models` defines Pydantic DTOs for inter-service communication (both REST and event schemas). `eusolicit-kraftdata` defines type-safe request/response models for the KraftData external API.

**Acceptance Criteria -- eusolicit-models**:
- [ ] REST DTOs: `OpportunityDTO`, `AgentDTO`, `SubscriptionDTO`, `NotificationDTO`, `TaskDTO`, `TenantDTO` with full field definitions
- [ ] Event schemas: `OpportunitiesIngested`, `AgentCompleted`, `SubscriptionChanged`, `TaskAssigned`, `NotificationRequested`, `BillingEvent` -- each with `event_type` literal discriminator
- [ ] All event schemas extend a `BaseEvent` with `event_id`, `timestamp`, `correlation_id`, `source_service`, `tenant_id`
- [ ] Enums: `OpportunityStatus`, `AgentStatus`, `TaskPriority`, `NotificationChannel`, `SubscriptionTier`
- [ ] Unit tests validating serialization/deserialization round-trips for all DTOs and events

**Acceptance Criteria -- eusolicit-kraftdata**:
- [ ] Request models: `AgentRunRequest`, `WorkflowRunRequest`, `TeamRunRequest`, `StorageResourceRequest`, `WebhookRegistrationRequest`
- [ ] Response models: `AgentRunResponse`, `WorkflowRunResponse`, `TeamRunResponse`, `StorageResourceResponse`, `WebhookEvent`
- [ ] Status enums: `RunStatus` (pending, running, completed, failed), `ResourceType`, `WebhookEventType`
- [ ] All models include docstrings referencing KraftData API documentation
- [ ] Types only -- no HTTP client code, no `httpx`/`requests` dependency
- [ ] Unit tests for model instantiation and validation

**Implementation Notes**: `eusolicit-models` event schemas should use `Literal` type discriminators so consumers can pattern-match on `event_type`. KraftData models should match the external API spec as closely as possible -- use `Field(alias=...)` where the API uses different naming conventions.

---

### S01.08: GitHub Actions CI Pipeline

**Points**: 3 | **Type**: devops
**Description**: Create a GitHub Actions workflow that runs on every push and PR. It should lint, type-check, and test all 5 services and 3 shared packages in a matrix build. Include caching for pip dependencies and Docker layers.

**Acceptance Criteria**:
- [ ] Workflow triggers on push to `main` and on all pull requests
- [ ] Matrix strategy builds all 5 services and 3 packages (8 jobs) in parallel
- [ ] Each job runs: `ruff check .` (lint), `mypy .` (type check), `pytest` (unit tests)
- [ ] `ruff` config is defined once at the repo root (`ruff.toml` or `pyproject.toml`) and shared by all projects
- [ ] `mypy` config is defined once at the repo root (`mypy.ini` or `pyproject.toml`) with per-package overrides as needed
- [ ] pip dependencies are cached between runs (keyed by `pyproject.toml` hash)
- [ ] Docker layer caching is enabled for integration tests that need Docker Compose
- [ ] Failed jobs report which specific check failed (lint vs type vs test) in the GitHub PR status
- [ ] Workflow completes in under 10 minutes for a clean run

**Implementation Notes**: Use `actions/setup-python@v5` with Python 3.12. Use `pip install -e ".[dev]"` for each project to pick up dev dependencies. Consider a reusable workflow (`.github/workflows/python-check.yml`) called by the main CI workflow to reduce duplication. `ruff` should enforce `I` (isort), `E`/`W` (pycodestyle), `F` (pyflakes), and `UP` (pyupgrade) rule sets at minimum.

---

### S01.09: Helm Chart Base Template & Service Values

**Points**: 3 | **Type**: devops
**Description**: Create a reusable Helm chart that all 5 services share as a base, plus per-service `values.yaml` files that customize it. The chart should produce Deployment, Service, ConfigMap, HPA, and Ingress resources.

**Acceptance Criteria**:
- [ ] Base chart at `infra/helm/eusolicit-service/` with `Chart.yaml`, `templates/`, and `values.yaml`
- [ ] Templates include: `deployment.yaml`, `service.yaml`, `configmap.yaml`, `hpa.yaml`, `ingress.yaml`, `serviceaccount.yaml`
- [ ] Deployment template supports: resource requests/limits, liveness/readiness probes pointing to `/healthz`, environment variables from ConfigMap and Secrets, volume mounts
- [ ] HPA template supports min/max replicas and CPU/memory target utilization
- [ ] Per-service values files at `infra/helm/values/<service-name>.yaml` for all 5 services
- [ ] Each values file sets service-specific: image name, port, resource limits, replica count, environment variables
- [ ] `helm template` renders valid YAML for each service without errors
- [ ] Chart supports optional `podDisruptionBudget` and `networkPolicy` via feature flags in values

**Implementation Notes**: Use Helm 3 conventions. The base chart should be generic enough to deploy any of the 5 services by just swapping the values file. Use `helm template --debug` during development to catch rendering issues. Probes should have configurable paths and initial delay.

---

### S01.10: Terraform Scaffold & Placeholder Modules

**Points**: 2 | **Type**: devops
**Description**: Create a Terraform project structure with placeholder modules for the cloud resources the platform will need. This sprint only creates the scaffold -- actual resource provisioning is deferred.

**Acceptance Criteria**:
- [ ] Root at `infra/terraform/` with `main.tf`, `variables.tf`, `outputs.tf`, `providers.tf`, `backend.tf`
- [ ] Module directories (with placeholder `main.tf`, `variables.tf`, `outputs.tf` each): `modules/networking`, `modules/kubernetes`, `modules/database`, `modules/redis`, `modules/storage`, `modules/monitoring`
- [ ] Environment directories: `environments/dev/`, `environments/staging/`, `environments/prod/` each with `terraform.tfvars` and `main.tf` that references modules
- [ ] `providers.tf` configures AWS provider (or cloud-agnostic placeholder) with version constraints
- [ ] `backend.tf` configures S3 backend for remote state (with placeholder bucket name)
- [ ] `terraform validate` passes on the root module
- [ ] A README in `infra/terraform/` explains the module structure and intended usage

**Implementation Notes**: Keep modules minimal -- just input variables and empty resource blocks with TODO comments describing what will be provisioned. The goal is to establish the directory structure and module interface contracts, not to provision anything. Use Terraform >= 1.5 syntax.

## Technical Notes

- **Python version**: 3.12+ across all services. Use modern syntax (`type` statements, `match/case` where appropriate).
- **Dependency management**: Each service's `pyproject.toml` declares its own dependencies. Shared packages use relative path references (`eusolicit-common = {path = "../../packages/eusolicit-common", develop = true}`).
- **Testing convention**: `pytest` with `pytest-asyncio` for async tests, `pytest-cov` for coverage. Test files mirror source structure under `tests/`.
- **Code style**: `ruff` for linting and formatting (replaces black, isort, flake8). Single config at repo root.
- **Logging**: All services MUST use `structlog` via `eusolicit-common`. No direct `print()` or `logging.getLogger()` calls.
- **Environment variables**: All config via env vars. No hardcoded connection strings. The config loader in `eusolicit-common` is the single source of truth.
- **Docker images**: Multi-stage builds. Stage 1: install dependencies. Stage 2: copy source. Final image based on `python:3.12-slim`. Target image size < 200MB per service.
- **Event bus design**: Redis Streams chosen over Kafka for operational simplicity at current scale. The `EventPublisher`/`EventConsumer` abstraction allows swapping to Kafka later without changing service code.
- **Schema isolation**: PostgreSQL schemas (not separate databases) are used for logical isolation while allowing cross-schema JOINs on `shared` tables when needed via the `migration_role`.
