# EU Solicit — Project Delivery Instructions

Project-specific delivery rules for **eusolicit**. Read in conjunction with
the global cross-project rules. The global rules state principles
(constant-time compare, respect documented boundaries, use the project's
authorization layer); this file pins them to the concrete eusolicit
implementation.

Application code lives under `eusolicit-app/`. All commands below run from
that directory unless noted.

For broader project context (architecture, service inventory, full schema
map) see `eusolicit-docs/project-context.md` and the project's `CLAUDE.md`.

---

## Definition of done — eusolicit commands

A story is not complete until the following pass on the changed surface:

- `make lint` (ruff)
- `make type-check` (mypy)
- The right test target:
  - `make test-unit` — pure logic, no external deps
  - `make test-integration` — needs `make infra` (postgres + redis up)
  - `make test-api` — needs all services running
  - `make test-service SVC=<name>` — single service
- `make coverage` — minimum **80%** line coverage; HTML at `htmlcov/index.html`
- Frontend changes (from `frontend/`): `pnpm lint && pnpm type-check`,
  plus `pnpm check:i18n` if any user-facing string was added.
- E2E if you touched a user flow: `make test-e2e-chromium` at minimum.

If `make` complains about missing infra, run `make infra` first, then
`make migrate-all` after a `make reset-db`.

---

## Database & schema rules

One Postgres database, six schemas: **`client`**, **`admin`**, **`pipeline`**,
**`gateway`**, **`notification`**, **`shared`**. Each service role has CRUD
on **its own schema only**; `migration_role` owns DDL.

- **Never** join across schemas in application code. If two services need
  the same data, expose it via API or via Redis Streams events — not via
  SQL across schemas. There is no exception to this rule.
- New tables go in the owning service's schema. If a table is genuinely
  cross-cutting, place it in `shared` and update grants explicitly.
- Migrations are Alembic. After `make reset-db` you **must** run
  `make migrate-all` (dependency order matters).

## Authorization (RBAC)

- Endpoint-level company access goes through `check_entity_access()` —
  the dependency factory in `eusolicit-app/services/client-api/client_api/core/rbac.py`.
  Do not invent ad-hoc permission checks inline.
- Company roles: `admin`, `bid_manager`, `contributor`, `reviewer`,
  `read_only`. `EntityPermission` records can override the role ceiling
  per entity — honor whichever is more restrictive for the operation.
- Every cross-tenant endpoint needs a negative test: company A requests
  company B's resource → expect **403**. One positive test is not enough.

## Security must-dos (eusolicit-specific)

- Compare HMAC / Stripe webhook signatures with `hmac.compare_digest()` —
  never `==`.
- Every auth-touching path checks `User.is_active` before authorizing.
- Every `httpx` call sets an explicit `timeout=`. No exceptions for
  KraftData, Stripe, Google, or any internal call.
- Never log JWTs, OAuth codes, Stripe keys, session tokens, or PII —
  even at DEBUG.
- Webhook handlers must verify signatures before reading the payload.

## Testing — fixtures & markers

Use the right pytest marker; the Makefile targets are wired to these:

- `@pytest.mark.unit` — pure logic, no I/O
- `@pytest.mark.integration` — needs postgres + redis (`make infra`)
- `@pytest.mark.api` — needs running services
- `@pytest.mark.cross_service` — full stack

The root `tests/conftest.py` provides:
`db_session`, `clean_redis`, `client_api` / `admin_api` / `ai_gateway`
httpx clients, `UserFactory`, `CompanyFactory`, `OpportunityFactory`,
`register_and_verify_with_role`, `create_company_pair`. Use these instead
of hand-rolling test setup.

Isolation rules:

- **Never** call `commit()` inside a test. The `db_session` fixture
  handles per-test transaction rollback. Committing breaks isolation for
  every parallel test.
- The `clean_redis` fixture flushes **DB 1**; the application uses **DB 0**.
  Never write test data to DB 0.
- Service-level tests must override `get_db_session` and `get_redis_client`
  via `app.dependency_overrides`, then clear overrides in `finally`.

## Service & code conventions

- Python **3.12**, async throughout. Every module starts with
  `from __future__ import annotations`.
- Logging via `structlog` only — no `print`, no stdlib `logging` in
  service code.
- Each service subclasses `BaseServiceSettings` (from `eusolicit-common`)
  with its own `env_prefix`: `CLIENT_API_`, `ADMIN_API_`, `DATA_PIPELINE_`,
  `AI_GATEWAY_`, `NOTIFICATION_`, `INTEGRATIONS_API_`.
- Read settings via `get_settings()`. Never reach into `os.environ`
  inside business logic.
- `eusolicit-app/ruff.toml`: line length **120**, rules `I E W F UP`.
  `__init__.py` may keep `F401` re-exports.

## Service ports (don't hardcode in tests — use clients from conftest)

| Service | Port |
|---|---|
| client-api | 8001 |
| admin-api | 8002 |
| data-pipeline | 8003 |
| ai-gateway | 8004 |
| notification | 8005 |
| integrations-api | 8007 |
| Next.js client | 3000 |
| Next.js admin | 3001 |

## Frontend conventions (Turborepo + Next.js 14 App Router)

- Data fetching: TanStack Query v5, **always** wrapped in `<QueryGuard>`.
- Forms: `useZodForm` + `<FormField>` (RHF + Zod). No bare `useForm`
  in new code.
- State: Zustand with the project's namespaced persist keys —
  `eusolicit-client-auth-store`, `eusolicit-admin-auth-store`. Don't
  invent new persist keys without coordinating.
- i18n: `next-intl`. Adding a string without updating all locale files
  fails `pnpm check:i18n`.
- Rich text: TipTap. Don't introduce a second editor.

---

## When in doubt (eusolicit-specific reflexes)

- A change crossing two services: prefer a Redis Streams event over a
  synchronous HTTP call between services.
- A non-trivial migration (data backfill, NOT NULL on a populated column,
  FK addition, index rebuild on a large table): call it out **before**
  applying — these usually need a multi-step plan.
- A KraftData ingestion change: Celery tasks live in `data-pipeline`; do
  not inline KraftData calls in `client-api`.
- An AI-agent execution change: route through `ai-gateway`'s registry
  (`config/agents.yaml`), do not call providers directly from
  `client-api`.
