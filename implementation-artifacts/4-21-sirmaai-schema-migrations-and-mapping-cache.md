# Story 4.21: SirmaAI Schema Migrations + Tenant↔Project Mapping Cache

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer landing the second commit of the SirmaAI platform pivot inside Epic 4 (S04.20 renamed the surface; this story lays the persistence + cache foundation everything else in the amendment leans on)**,
I want **two Alembic migrations (one in `client-api`, one in `sirmaai-gateway`) that create the three net-new tables defined in architecture amendment §4.1 — `client.sirmaai_projects` (with `agent_map` JSONB + Fernet-encrypted `api_key_encrypted` BYTEA + partial index on non-`provisioned` rows), `gateway.webhook_subscriptions` (HMAC secret store, Fernet-encrypted), `gateway.workflow_runs` (with partial index on `status IN ('pending','running')`) — together with their SQLAlchemy ORM models, a Redis-backed mapping cache for `client.sirmaai_projects` (5-minute TTL, key `sirmaai:project:by-company:{company_id}`, invalidated on the `sirmaai.key_rotated` event published in S04.22), a `sirmaai.key_rotated` Redis-Streams consumer that performs the invalidation, schema-isolation invariant verification (ADR-001 — no FKs out of `gateway`, only `client.companies` FK out of `client.sirmaai_projects` / `gateway.workflow_runs`), and a negative-path test for "missing tenant → 404 / `TenantNotProvisionedError`"**,
so that **S04.22 (api-key vault + rotation), S04.23 (logical-name resolution via `agent_map`), S04.24 (async-run + jobs polling — writes `gateway.workflow_runs` rows), S04.25 (Standard Webhooks receiver — reads/writes `gateway.webhook_subscriptions` HMAC secret), S04.26 (run-state reconciler — scans the `gateway.workflow_runs` partial index every 5 minutes), and E24 (tenant provisioning — inserts the `client.sirmaai_projects` row at provisioning time) all have the schema, ORM, cache, and event-driven invalidation contract they need waiting for them when they start — and so the cache-hit-rate ≥99% steady-state SLO from the amendment AC list can be met from day one without any consumer needing to roll its own per-company lookup**.

## Acceptance Criteria

1. **client-api Alembic migration `072_create_sirmaai_projects.py`** creates `client.sirmaai_projects` with the exact column shape, constraints, and partial index from architecture amendment §4.1 lines 342–358:
   - `id UUID PRIMARY KEY DEFAULT gen_random_uuid()` (server_default `sa.text("gen_random_uuid()")` per project convention; ORM default also `uuid.uuid4`).
   - `company_id UUID NOT NULL UNIQUE REFERENCES client.companies(id) ON DELETE CASCADE` (cascade matches `crm_connections` pattern at `services/client-api/src/client_api/models/crm_connection.py`).
   - `sirmaai_org_id TEXT NOT NULL` — the singleton EU Solicit Org id (one value cluster-wide; not unique because every row carries it).
   - `sirmaai_project_id TEXT NOT NULL UNIQUE` — the per-company SirmaAI Project id.
   - `api_key_encrypted BYTEA NOT NULL` — Fernet ciphertext from `eusolicit_common.crypto.FernetCrypto.encrypt()` (Epic 9 canonical module at `packages/eusolicit-common/src/eusolicit_common/crypto.py`).
   - `api_key_rotated_at TIMESTAMPTZ NOT NULL DEFAULT now()`.
   - `n8n_subdomain TEXT NULL` (nullable; org-shared but project-routed per architecture amendment §3.4).
   - `agent_map JSONB NOT NULL DEFAULT '{}'::jsonb` — `{logical_name: sirmaai_agent_uuid}` adapter map.
   - `provisioning_status TEXT NOT NULL DEFAULT 'pending'` with `CHECK (provisioning_status IN ('pending','provisioned','failed','archived'))`.
   - `provisioning_error TEXT NULL`.
   - `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `archived_at TIMESTAMPTZ NULL`.
   - **Partial index** `ix_sirmaai_projects_status ON client.sirmaai_projects(provisioning_status) WHERE provisioning_status != 'provisioned'` — hot-row index for the provisioner's "show me what's still being set up" query, exactly per amendment.
   - Migration revision id `072`, `down_revision = "071"` (current head after `071_create_nps_responses.py`), `branch_labels = None`, `depends_on = None`.
   - Reversible: `downgrade()` drops the partial index then the table.

2. **sirmaai-gateway Alembic migration `004_sirmaai_webhook_and_workflow_run_tables.py`** creates two tables in the `gateway` schema, idempotent against the existing `001_initial.py`, `002_webhook_log.py`, `003_agent_executions.py` chain:
   - `gateway.webhook_subscriptions`:
     - `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`.
     - `sirmaai_subscription_id TEXT NOT NULL UNIQUE`.
     - `event_types TEXT[] NOT NULL` (Postgres array).
     - `hmac_secret_encrypted BYTEA NOT NULL` — Fernet-encrypted secret used by S04.25's HMAC verifier (constant-time `hmac.compare_digest()`).
     - `hmac_rotated_at TIMESTAMPTZ NOT NULL DEFAULT now()`.
     - `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`.
   - `gateway.workflow_runs`:
     - `id UUID PRIMARY KEY DEFAULT gen_random_uuid()`.
     - `company_id UUID NOT NULL REFERENCES client.companies(id)` — **the only cross-schema FK out of `gateway`** (architecture amendment §4.1 line 437; ADR-001 carve-out for tenant identity). No `ON DELETE` cascade — orphan workflow_runs are kept for forensics; cleanup happens via tombstone in a separate retention job.
     - `eusolicit_run_id UUID NOT NULL UNIQUE` — internal correlation id (callers pass this in via `X-EU-Solicit-Run-Id`; if absent, gateway generates).
     - `sirmaai_run_id TEXT NULL` (null until SirmaAI assigns).
     - `sirmaai_job_id TEXT NULL` (async-run jobs returned by S04.24).
     - `run_type TEXT NOT NULL` with `CHECK (run_type IN ('agent','team','workflow'))`.
     - `status TEXT NOT NULL DEFAULT 'pending'` with `CHECK (status IN ('pending','running','completed','failed','cancelled'))`.
     - `started_at TIMESTAMPTZ NOT NULL DEFAULT now()`.
     - `completed_at TIMESTAMPTZ NULL`.
     - `last_polled_at TIMESTAMPTZ NULL` — reconciler poll cursor.
     - `error_message TEXT NULL`.
     - `payload_excerpt JSONB NULL` — **bounded to 16 KiB serialised** via DB-side `CHECK (octet_length(payload_excerpt::text) <= 16384)`; redacted breadcrumb only (full traces live in SirmaAI).
     - **Partial index** `ix_workflow_runs_nonterminal ON gateway.workflow_runs(last_polled_at) WHERE status IN ('pending','running')` — the index the 5-minute reconciler from S04.26 scans.
   - Migration revision id `004`, `down_revision = "003"`, `branch_labels = None`, `depends_on = None`.
   - Reversible: `downgrade()` drops partial index, both tables, and the CHECK constraints.

3. **SQLAlchemy ORM models match the migrations exactly** and are registered with the right `Base.metadata` so future `alembic autogenerate` runs see them:
   - `services/client-api/src/client_api/models/sirmaai_project.py` defines `class SirmaAIProject(Base)` with `__tablename__ = "sirmaai_projects"`, `__table_args__ = (..., sa.CheckConstraint(...), {"schema": "client"})`. Columns typed via `Mapped[...]` with the same `mapped_column(...)` pattern used in `crm_connection.py`. `JSONB` via `sqlalchemy.dialects.postgresql.JSONB`. `BYTEA` via `sa.LargeBinary`. Exported from `client_api/models/__init__.py` so `Base.metadata` knows about it.
   - `services/sirmaai-gateway/src/sirmaai_gateway/models/webhook_subscription.py` defines `class WebhookSubscription(Base)` with `__tablename__ = "webhook_subscriptions"`, `__table_args__ = (..., {"schema": "gateway"})`, `event_types: Mapped[list[str]] = mapped_column(sa.ARRAY(sa.Text), nullable=False)`.
   - `services/sirmaai-gateway/src/sirmaai_gateway/models/workflow_run.py` defines `class WorkflowRun(Base)` with `__tablename__ = "workflow_runs"`, FK to `client.companies(id)`, both CHECK constraints, `payload_excerpt` as `Mapped[dict | None] = mapped_column(JSONB, nullable=True)`. Exported from `sirmaai_gateway/models/__init__.py` so `Base.metadata` registers it.
   - **No relationship() back-refs across schemas.** The cross-schema FK on `gateway.workflow_runs.company_id` is declared as a column-level `sa.ForeignKey("client.companies.id")` only — no ORM `relationship()` to `Company`. (ADR-001 schema isolation: gateway code must not navigate to client objects via ORM.)

4. **Redis-backed mapping cache module** at `services/sirmaai-gateway/src/sirmaai_gateway/services/project_cache.py` exposes:
   - `class ProjectCache` with constructor `(redis_client: redis.asyncio.Redis, session_factory: async_sessionmaker[AsyncSession], crypto: FernetCrypto, ttl_seconds: int = 300)`.
   - `async def get(self, company_id: uuid.UUID) -> ProjectMapping` returning a Pydantic model `ProjectMapping(sirmaai_project_id: str, sirmaai_org_id: str, api_key_plaintext: str, agent_map: dict[str, str], n8n_subdomain: str | None, provisioning_status: str)`.
   - Cache key: `f"sirmaai:project:by-company:{company_id}"` (NB: this is on the application's Redis DB 0; test isolation per `clean_redis` fixture covers DB 1, never DB 0).
   - Cache miss path: `SELECT * FROM client.sirmaai_projects WHERE company_id = :company_id` — **but** `sirmaai-gateway` has CRUD on the `gateway` schema, not `client`. Use a **read-only** asyncpg/SQLAlchemy connection that the `ai_gateway_role` already has via `GRANT SELECT ON ALL TABLES IN SCHEMA client TO ai_gateway_role`. **VERIFY THIS GRANT EXISTS before relying on it** — see Task 6 below; if it doesn't, the migration must add it.
   - Cache hit path: deserialise the cached payload (JSON), then **decrypt `api_key_encrypted` via `FernetCrypto.decrypt()`** before returning. (Storing the plaintext key in Redis would mirror the at-rest leak we're trying to prevent — only cache the ciphertext bytes, decrypt on every cache read; the perf cost is negligible and the security posture is correct.)
   - Cache value shape: JSON-encoded dict with `b64(api_key_encrypted)` for the BYTEA bytes (Fernet ciphertext is already URL-safe base64 — store the raw bytes via base64 string).
   - `async def invalidate(self, company_id: uuid.UUID) -> None` → `await redis.delete(key)`.
   - `async def invalidate_all(self) -> None` → `SCAN` for `sirmaai:project:by-company:*` and `UNLINK` them (used by the bootstrap and the "operator-forced flush" admin endpoint added incidentally — admin endpoint is out of scope; just expose the method).
   - **Missing-tenant behaviour**: cache miss + no DB row → raise `TenantNotProvisionedError(company_id)` (a new typed exception in `services/sirmaai-gateway/src/sirmaai_gateway/services/exceptions.py`). **The cache does NOT cache the negative result** — caching "tenant doesn't exist" with a 5-minute TTL would mean a freshly-provisioned tenant could be locked out for ≤5 minutes after E24 inserts the row; instead, the caller is expected to surface the error to the user (S04.23 turns it into HTTP 404; consumer-facing endpoints already in client-api map it to a "Tenant not provisioned yet — retry in a moment" 503).
   - Connection pooling: reuse the engine from `sirmaai_gateway/services/db.py` (one engine per service instance); do not create a new engine per call. Explicit `httpx`/asyncpg timeout: query timeout `5s` (statement_timeout via `SET LOCAL`) — the cache should fail fast rather than hold the request.

5. **Cache invalidation via Redis-Streams event consumer.** A new consumer in `services/sirmaai-gateway/src/sirmaai_gateway/services/cache_invalidation_consumer.py`:
   - Uses `eusolicit_common.events.EventConsumer` (the shared package; do not roll your own xreadgroup loop).
   - Subscribes to stream `sirmaai:gateway:key-rotated` with consumer group `sirmaai-gateway-cache-invalidation` and consumer name `f"{settings.service_name}-{hostname}"`.
   - On every `sirmaai.key_rotated` event (payload schema: `{"company_id": "<uuid>", "rotated_at": "<iso8601>"}` per architecture amendment §5.3), call `ProjectCache.invalidate(company_id)`.
   - Started in `main.py` `lifespan` startup as an `asyncio.create_task(...)` BEHIND THE FLAG `if settings.sirmaai_gateway_enabled:` — when the flag is `False`, the consumer does NOT start (so the pre-amendment KraftData code path is bit-identical to today). Task is cancelled cleanly on shutdown via `try: ... finally: task.cancel()`.
   - Idempotent: if `invalidate()` runs against a key that doesn't exist in Redis (race with TTL expiry), it is a no-op (`DEL` returns `0`, no exception).
   - Consumer group is bootstrap-created via `bootstrap_event_bus()` in `eusolicit_common.events.bootstrap` — register `sirmaai:gateway:key-rotated` in the `STREAMS` constant and `sirmaai-gateway-cache-invalidation` in `CONSUMER_GROUPS` (single-line additions; both constants are dicts in `bootstrap.py`).

6. **Schema-grant audit + cross-schema SELECT grant.** Before/after the migration, the developer must run `\dp client.sirmaai_projects` (psql) or equivalent ORM query and confirm `ai_gateway_role` has `SELECT` on `client.companies` AND on `client.sirmaai_projects`. The existing init script at `eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql` grants each service role CRUD on its own schema only — by default `ai_gateway_role` cannot SELECT `client.companies`. **If the grant does not already exist**, the `client-api` migration `072` MUST add it as a DDL step at the end of `upgrade()`:
   ```sql
   GRANT SELECT ON client.companies TO ai_gateway_role;
   GRANT SELECT ON client.sirmaai_projects TO ai_gateway_role;
   ALTER DEFAULT PRIVILEGES IN SCHEMA client GRANT SELECT ON TABLES TO ai_gateway_role;
   ```
   The grant block is idempotent (Postgres `GRANT` is). Verify via the negative test below.
   **NB:** This is a deliberate, ADR-001-compatible carve-out: `ai_gateway_role` gets **SELECT-only** access to `client.companies` and `client.sirmaai_projects` — never INSERT/UPDATE/DELETE. The grant is documented in the migration's docstring and called out in the existing `test_db_schema_isolation.py` test (extend it — see Task 8).

7. **Negative-path tests** in `services/sirmaai-gateway/tests/integration/test_project_cache.py`:
   - `test_missing_tenant_raises_tenant_not_provisioned_error` — set up two companies A and B in `client.companies` via `eusolicit_test_utils` factories; insert `client.sirmaai_projects` row for company A only; assert `await cache.get(B.id)` raises `TenantNotProvisionedError(company_id=B.id)`. **No negative caching** — verify by calling `cache.get(B.id)` twice and confirming both calls execute the SQL (via `respx`-style asyncpg query counter or `unittest.mock.patch` on the session factory).
   - `test_cache_hit_after_initial_db_read` — `cache.get(A.id)` twice; first call hits DB, second is `< 5ms` and does NOT execute SQL (assert via mock). Verifies the cache layer actually caches.
   - `test_invalidate_on_key_rotated_event` — publish a `sirmaai.key_rotated` event for company A via `EventPublisher`; assert the cache entry is gone within `200ms` (poll-with-timeout pattern, NOT `await asyncio.sleep(1)`). Verifies the consumer wiring.
   - `test_api_key_decrypted_on_cache_read` — store a Fernet-encrypted key for company A, `cache.get(A.id)` returns the **plaintext** key in `ProjectMapping.api_key_plaintext`; the raw Redis cached value (peek via `redis.get(key)`) contains base64'd ciphertext, NOT plaintext. Verifies decryption boundary.
   - `test_cross_tenant_lookup_returns_correct_row` — company A and B both have rows; `cache.get(A.id)` returns A's `sirmaai_project_id`, `cache.get(B.id)` returns B's; this is the equivalent of the cross-tenant negative test pattern from the project's delivery instructions (one positive is not enough).

8. **Schema-isolation invariant test** extends `eusolicit-app/tests/integration/test_db_schema_isolation.py` (existing pattern there — see lines 45–62 for the role-schema map):
   - Add `gateway.workflow_runs` and `gateway.webhook_subscriptions` to the table iteration; verify `ai_gateway_role` has CRUD on both.
   - Add `client.sirmaai_projects` to the role-table negative grid: verify `ai_gateway_role` has **SELECT** (per AC #6) but NOT `INSERT/UPDATE/DELETE` on `client.sirmaai_projects`; verify `client_api_role` has full CRUD on `client.sirmaai_projects`; verify `notification_role` / `data_pipeline_role` / `admin_api_role` have NO access (expect `permission denied` from asyncpg).
   - Verify no FKs out of `gateway` other than the `client.companies` FK on `gateway.workflow_runs` (query `information_schema.referential_constraints` filtered by the `gateway` schema).

9. **ORM round-trip tests** in `services/sirmaai-gateway/tests/unit/test_workflow_run_model.py` and `services/client-api/tests/unit/test_sirmaai_project_model.py`:
   - Insert a `SirmaAIProject` with `agent_map={"proposal_drafter": "uuid-abc", "espd_filler": "uuid-def"}`, retrieve, assert JSONB round-trip preserves dict semantics.
   - Insert a `SirmaAIProject` with `provisioning_status="invalid"` — expect SQLAlchemy `IntegrityError` (CHECK constraint).
   - Insert a `WorkflowRun` with `run_type="invalid"` — expect `IntegrityError` (CHECK constraint).
   - Insert a `WorkflowRun` with `payload_excerpt={"data": "x" * 20000}` (>16KiB) — expect `IntegrityError` (octet_length CHECK).
   - Insert a `SirmaAIProject` with `api_key_encrypted=FernetCrypto(test_key).encrypt("secret")`, retrieve, assert `FernetCrypto(test_key).decrypt(row.api_key_encrypted) == "secret"` (verifies the BYTEA column round-trips Fernet ciphertext intact).
   - Use the project's standard `db_session` fixture (per-test transaction rollback); **never `await session.commit()`** inside any test (project rule).

10. **Cache-hit-rate observability.** The `ProjectCache` emits two Prometheus counters via the existing `eusolicit_common.observability.metrics` module:
   - `sirmaai_project_cache_hits_total{service="sirmaai-gateway"}`
   - `sirmaai_project_cache_misses_total{service="sirmaai-gateway"}`
   These are scraped by the existing Prometheus job `sirmaai-gateway` (configured in S04.20). Grafana panel will be added by S04.27 / a later observability story; the metrics simply need to be emitted with the right labels. **Acceptance check**: `await cache.get(company_id)` increments the right counter; verify in a unit test using `prometheus_client.REGISTRY.get_sample_value(...)`. The amendment AC "cache-hit rate ≥99% in steady state" is met operationally; this story just makes it measurable.

11. **Feature-flag respect.** The DDL migrations land unconditionally (they're additive and reversible). But **all runtime code paths** wiring the `ProjectCache` into request handlers MUST stay behind `if settings.sirmaai_gateway_enabled:` — when the flag is `False`, the cache is **not constructed**, the consumer is **not started**, and no request path reads `client.sirmaai_projects`. This preserves the S04.20 invariant that pre-amendment code paths are bit-identical until the flag flips. Concretely:
   - `main.py` `lifespan` startup: `if settings.sirmaai_gateway_enabled: app.state.project_cache = ProjectCache(...)` — else `app.state.project_cache = None`.
   - The cache-invalidation consumer task is `asyncio.create_task(...)` only when the flag is `True`.
   - No production code in `routers/` reads `app.state.project_cache` yet (S04.23+ wire that); this story only constructs and lifecycles it.

12. **Settings additions on `SirmaAIGatewaySettings`** (`src/sirmaai_gateway/config.py`):
   - `sirmaai_org_id: str = ""` — the singleton EU Solicit Org id (set per-env; in dev points at the shared staging Org per S04.31).
   - `sirmaai_fernet_key: str = ""` — base64 Fernet key for `api_key_encrypted` / `hmac_secret_encrypted` columns. Same key shape as `CALENDAR_ENCRYPTION_KEY` / `CRM_TOKEN_ENCRYPTION_KEY` (Epic 9 canonical). **Must be set when `sirmaai_gateway_enabled=True`**: `main.py` lifespan asserts `len(settings.sirmaai_fernet_key) > 0` raises `RuntimeError("SIRMAAI_FERNET_KEY required when SIRMAAI_GATEWAY_ENABLED=true")` so dev catches misconfig fast.
   - `project_cache_ttl_seconds: int = 300` (5 minutes; matches amendment AC).
   - All three read from env vars: `SIRMAAI_ORG_ID`, `SIRMAAI_FERNET_KEY`, `PROJECT_CACHE_TTL_SECONDS`.
   - Update `.env.example` at repo root: append the three new keys with one-line comments referencing this story.
   - **NB**: `SIRMAAI_FERNET_KEY` is **secret material** — never log it, never include in `/health` or `/admin/*` responses, never include in `app.openapi()` schemas. The settings class field stays a plain `str`; rely on the logging discipline (project rule: "never log secrets, JWTs, API keys, …").

13. **No regression** — full existing sirmaai-gateway and client-api test suites stay green. After this story:
   - `make test-service SVC=sirmaai-gateway` passes; coverage on `sirmaai_gateway/services/` stays ≥85% (the new cache module contributes substantial coverage; the pre-existing 83.36% gap from S04.20 known deviation is documented but should improve, not worsen).
   - `make test-service SVC=client-api` passes; coverage on `client_api/models/` is unaffected (new model is small).
   - `make test-integration` passes; new tests in `test_db_schema_isolation.py` and `test_project_cache.py` are included.
   - `make migrate-all` succeeds locally after `make reset-db`; both new migrations apply cleanly in dependency order (`client-api` before `sirmaai-gateway` because `gateway.workflow_runs.company_id` FKs to `client.companies`).
   - `make lint` clean; `make type-check` clean (no new mypy errors in `sirmaai_gateway` or `client_api`).

14. **Migration safety call-out per project delivery instructions** ("Migration discipline" section):
   - **NOT NULL on populated columns?** No — both tables are net-new, no backfill required.
   - **Long-running rewrite?** No — `CREATE TABLE` + partial indexes are O(1) at table creation time.
   - **FK addition on populated table?** Yes — `gateway.workflow_runs.company_id REFERENCES client.companies(id)` adds an FK to a populated table. **The FK is added at `CREATE TABLE` time on an empty table** (the new table has no rows), so the lock duration is bounded by the constraint check on zero rows = near-instant. No `NOT VALID` / `VALIDATE CONSTRAINT` two-step is needed.
   - **Rollback plan**: `alembic downgrade -1` reverses both migrations cleanly. The Redis cache module + consumer have no schema state; they fail-fast at startup if the tables don't exist, so a downgrade on a running service requires either flipping the flag back to `False` first or restarting the gateway after the downgrade.
   - **Cross-schema FK rationale documented in migration docstring**: the `gateway.workflow_runs → client.companies` FK is the second sanctioned exception to ADR-001 (the first is `client.crm_connections → client.client_workspaces`, in-schema). Tenant identity is shared; everything else is owned per-schema.

15. **Sprint-status surgical update.** `eusolicit-docs/implementation-artifacts/sprint-status.yaml` row `4-21-sirmaai-schema-migrations-and-mapping-cache: backlog` is flipped to `ready-for-dev` by the `bmad-create-story` workflow (this story); `dev-story` will flip it to `in-progress` on pickup; `code-review` flips to `done`. `epic-4: in-progress` row stays `in-progress` (amendment audit comment on line 173). No regeneration of unrelated rows. Per memory `project_sprint_status_file.md`: file is orchestrator-managed — surgical edits only.

## Tasks / Subtasks

- [ ] **Task 1: client-api migration `072_create_sirmaai_projects.py` (AC: 1, 6, 14)**
  - [ ] 1.1 Create file `eusolicit-app/services/client-api/alembic/versions/072_create_sirmaai_projects.py` modelled on `054_create_crm_stage_mappings.py` (closest peer pattern). `revision = "072"`, `down_revision = "071"`. Module docstring cites architecture amendment §4.1 and Story 4.21.
  - [ ] 1.2 `upgrade()` body: `op.create_table("sirmaai_projects", ..., schema="client")` with every column from AC #1. Use `sqlalchemy.dialects.postgresql.JSONB` for `agent_map`, `sa.LargeBinary` for `api_key_encrypted`, `sa.ARRAY(sa.Text)` not needed here (no arrays). `server_default=sa.text("gen_random_uuid()")` for `id`, `server_default=sa.text("now()")` for `*_at` columns, `server_default=sa.text("'{}'::jsonb")` for `agent_map`, `server_default=sa.text("'pending'")` for `provisioning_status`.
  - [ ] 1.3 Add CHECK constraint inline in `create_table`: `sa.CheckConstraint("provisioning_status IN ('pending','provisioned','failed','archived')", name="check_sirmaai_projects_status_valid")`.
  - [ ] 1.4 After `create_table`, `op.create_index("ix_sirmaai_projects_status", "sirmaai_projects", ["provisioning_status"], schema="client", postgresql_where=sa.text("provisioning_status != 'provisioned'"))` — Alembic supports partial indexes via `postgresql_where`.
  - [ ] 1.5 Add the GRANT block from AC #6 via `op.execute()` statements (three idempotent GRANTs). Wrap each in a comment explaining the ADR-001 SELECT-only carve-out.
  - [ ] 1.6 `downgrade()` body: `op.drop_index("ix_sirmaai_projects_status", table_name="sirmaai_projects", schema="client")`, then `op.drop_table("sirmaai_projects", schema="client")`. **Do NOT** revoke the GRANTs in downgrade — they're idempotent and revoking would risk leaving the role in a broken state if the downgrade is part of a rollback during an outage. Document this in the downgrade docstring.

- [ ] **Task 2: sirmaai-gateway migration `004_sirmaai_webhook_and_workflow_run_tables.py` (AC: 2, 14)**
  - [ ] 2.1 Create file `eusolicit-app/services/sirmaai-gateway/alembic/versions/004_sirmaai_webhook_and_workflow_run_tables.py`. `revision = "004"`, `down_revision = "003"`. Module docstring cites architecture amendment §4.1 + the cross-schema FK rationale.
  - [ ] 2.2 `upgrade()`: `op.create_table("webhook_subscriptions", ..., schema="gateway")` per AC #2 column list.
  - [ ] 2.3 `op.create_table("workflow_runs", ..., schema="gateway")` per AC #2 column list. Use `sa.ForeignKey("client.companies.id")` for the cross-schema FK (no `ondelete`). Two CHECK constraints inline: `check_workflow_runs_run_type_valid`, `check_workflow_runs_status_valid`, `check_workflow_runs_payload_excerpt_size` (`octet_length(payload_excerpt::text) <= 16384`).
  - [ ] 2.4 `op.create_index("ix_workflow_runs_nonterminal", "workflow_runs", ["last_polled_at"], schema="gateway", postgresql_where=sa.text("status IN ('pending','running')"))`.
  - [ ] 2.5 `downgrade()`: drop partial index, drop both tables (workflow_runs first since it has the FK, then webhook_subscriptions). CHECK constraints dropped implicitly with the tables.

- [ ] **Task 3: ORM models in client-api (AC: 3, 9)**
  - [ ] 3.1 Create `eusolicit-app/services/client-api/src/client_api/models/sirmaai_project.py` modelled on `crm_connection.py`. Columns typed via `Mapped[...]`. JSONB via `from sqlalchemy.dialects.postgresql import JSONB`. BYTEA via `sa.LargeBinary`. CHECK constraint in `__table_args__`. `{"schema": "client"}` last.
  - [ ] 3.2 Import + export in `client_api/models/__init__.py`: add `from .sirmaai_project import SirmaAIProject` next to `from .crm_connection import CrmConnection`. Also add `"SirmaAIProject"` to `__all__` if maintained.
  - [ ] 3.3 Verify `Base.metadata.tables["client.sirmaai_projects"]` exists at import time (smoke test in `tests/unit/test_sirmaai_project_model.py`).

- [ ] **Task 4: ORM models in sirmaai-gateway (AC: 3, 9)**
  - [ ] 4.1 Create `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/models/webhook_subscription.py`. Column types per AC #2. `__table_args__ = ({"schema": "gateway"},)`.
  - [ ] 4.2 Create `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/models/workflow_run.py`. FK column declared as `sa.ForeignKey("client.companies.id")` only — **no `relationship()`** back-ref. CHECK constraints in `__table_args__`. JSONB column for `payload_excerpt`.
  - [ ] 4.3 Update `sirmaai_gateway/models/__init__.py` to export `WebhookSubscription` and `WorkflowRun` alongside the existing `AgentExecution` and `WebhookLog`.

- [ ] **Task 5: `ProjectCache` mapping cache (AC: 4, 10, 11, 12)**
  - [ ] 5.1 Create `services/sirmaai-gateway/src/sirmaai_gateway/services/project_cache.py`. Pydantic `ProjectMapping(BaseModel)` model + `class ProjectCache` per AC #4. Use `structlog.get_logger(__name__)` (project rule: structlog only). All async; all `httpx`-like timeouts explicit (`asyncio.wait_for` with `5.0` on the SELECT).
  - [ ] 5.2 Create `services/sirmaai-gateway/src/sirmaai_gateway/services/exceptions.py` (or extend existing) with `class TenantNotProvisionedError(Exception)`; include `company_id` attribute; format `repr` for log lines (NEVER include the api-key plaintext in any error).
  - [ ] 5.3 Wire Prometheus counters per AC #10: register `sirmaai_project_cache_hits_total` and `sirmaai_project_cache_misses_total` via the existing `eusolicit_common.observability.metrics` registry helpers — copy the pattern from how the existing `agent_execution_total` / `rate_limit_rejected_total` counters are registered in the service.
  - [ ] 5.4 Add settings fields per AC #12 to `SirmaAIGatewaySettings` in `config.py`. Update `.env.example`. The `RuntimeError("SIRMAAI_FERNET_KEY required when SIRMAAI_GATEWAY_ENABLED=true")` assertion goes in `main.py` lifespan, gated by `if settings.sirmaai_gateway_enabled:`.
  - [ ] 5.5 In `main.py` lifespan startup: `if settings.sirmaai_gateway_enabled: app.state.project_cache = ProjectCache(redis_client, session_factory, FernetCrypto(settings.sirmaai_fernet_key), ttl_seconds=settings.project_cache_ttl_seconds)` — else `app.state.project_cache = None`. Cancel-clean on shutdown.

- [ ] **Task 6: Schema-grant verification + grant block in migration (AC: 6, 8)**
  - [ ] 6.1 Run psql against a clean `make reset-db` baseline: `\dp client.companies` and check if `ai_gateway_role=r/...` is listed. Document the finding in the migration docstring.
  - [ ] 6.2 If absent (expected — the init script grants per-schema only), include the three GRANT statements in `upgrade()` of `072_create_sirmaai_projects.py` per AC #6.
  - [ ] 6.3 If already present (unexpected), the GRANTs are still idempotent — leave them in for explicitness and document in the docstring.

- [ ] **Task 7: Cache-invalidation Redis-Streams consumer (AC: 5, 11)**
  - [ ] 7.1 Create `services/sirmaai-gateway/src/sirmaai_gateway/services/cache_invalidation_consumer.py`. Import `EventConsumer` from `eusolicit_common.events`. Define `async def run_cache_invalidation_consumer(cache: ProjectCache, redis_client, stop_event: asyncio.Event) -> None` that drives an `xreadgroup` loop via the shared consumer abstraction.
  - [ ] 7.2 Add the new stream + consumer-group constants to `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py`: extend the `STREAMS` dict with `{"sirmaai_gateway_key_rotated": "sirmaai:gateway:key-rotated"}` and the `CONSUMER_GROUPS` dict with `{"sirmaai-gateway-cache-invalidation": "sirmaai:gateway:key-rotated"}`. These constants are used by `bootstrap_event_bus()` on service startup to ensure the stream/group exists (`XGROUP CREATE … MKSTREAM`).
  - [ ] 7.3 In `main.py` lifespan, behind `if settings.sirmaai_gateway_enabled:`, after `app.state.project_cache` is built, spawn `app.state.cache_invalidation_task = asyncio.create_task(run_cache_invalidation_consumer(...))`. On shutdown, `app.state.cache_invalidation_task.cancel(); await asyncio.gather(app.state.cache_invalidation_task, return_exceptions=True)`.
  - [ ] 7.4 Event payload schema parsing: `payload["company_id"]` is a UUID string; cast via `uuid.UUID(payload["company_id"])` and propagate. Malformed events: log at WARN with the bad event id, skip, ACK the message (do NOT block the consumer-group offset — better to drop a malformed event than wedge the stream).

- [ ] **Task 8: Schema-isolation invariant tests (AC: 8)**
  - [ ] 8.1 Open `eusolicit-app/tests/integration/test_db_schema_isolation.py`. Add `"sirmaai_projects"` to the list of tables the test iterates for `client` schema (look for the existing `client.companies` / `client.users` iteration pattern). Add `"workflow_runs"` and `"webhook_subscriptions"` to the `gateway` schema iteration.
  - [ ] 8.2 Add a dedicated test `test_ai_gateway_role_has_select_only_on_sirmaai_projects` that connects as `ai_gateway_role` and asserts (a) `SELECT * FROM client.sirmaai_projects LIMIT 1` succeeds (empty result OK), (b) `INSERT INTO client.sirmaai_projects ...` raises `asyncpg.exceptions.InsufficientPrivilegeError`. Same for UPDATE/DELETE.
  - [ ] 8.3 Add a test `test_no_fks_out_of_gateway_except_companies` that queries `information_schema.referential_constraints + key_column_usage` filtered by `unique_constraint_schema NOT IN ('gateway', 'shared')` for any FK whose owning table is in `gateway`; assert the resulting set is exactly `{("workflow_runs", "client.companies.id")}` — no other cross-schema FKs.

- [ ] **Task 9: Negative-path + cache behaviour integration tests (AC: 7)**
  - [ ] 9.1 Create `services/sirmaai-gateway/tests/integration/test_project_cache.py`. Marker `@pytest.mark.integration`. Use the project's `db_session` and `clean_redis` fixtures from `tests/conftest.py` (NEVER commit; flush DB 1 only).
  - [ ] 9.2 Fixture: `provisioned_company` factory that inserts a `client.companies` row + matching `client.sirmaai_projects` row with `provisioning_status='provisioned'` and a Fernet-encrypted `api_key_encrypted`. Use the project `CompanyFactory` from `eusolicit-test-utils` if available; else inline insert via the `db_session`.
  - [ ] 9.3 Implement the five tests from AC #7 verbatim. The "no SQL on second call" assertion uses `unittest.mock.patch.object(session_factory, "begin")` — or wrap the session factory in a counting decorator. The "200ms invalidation" assertion uses an `async def poll(timeout=0.2):` helper, NOT `asyncio.sleep(1)`.
  - [ ] 9.4 The `test_invalidate_on_key_rotated_event` test publishes the event via `EventPublisher` (shared package); the consumer must already be running — bootstrap it in a per-test fixture that starts/stops the consumer task.

- [ ] **Task 10: ORM round-trip tests (AC: 9)**
  - [ ] 10.1 Create `services/client-api/tests/unit/test_sirmaai_project_model.py`. `@pytest.mark.integration` (model tests touch DB → integration tier, per project rule). Use `db_session` fixture.
  - [ ] 10.2 Cover: JSONB `agent_map` round-trip; CHECK constraint rejection (`provisioning_status="invalid"`); Fernet ciphertext round-trip (encrypt, store, fetch, decrypt — assert plaintext match).
  - [ ] 10.3 Create `services/sirmaai-gateway/tests/unit/test_workflow_run_model.py`. Cover: `run_type` CHECK; `status` CHECK; `payload_excerpt` size CHECK (insert 20KB blob → expect IntegrityError); cross-schema FK enforcement (insert with non-existent `company_id` → expect IntegrityError).

- [ ] **Task 11: Feature-flag gating verification (AC: 11)**
  - [ ] 11.1 Add a unit test `tests/unit/test_lifespan_flag_gating.py` that constructs the app with `SIRMAAI_GATEWAY_ENABLED=false` and asserts `app.state.project_cache is None` and no `cache_invalidation_task` attribute exists. Then constructs the app with `SIRMAAI_GATEWAY_ENABLED=true` + a valid `SIRMAAI_FERNET_KEY` and asserts both are present.
  - [ ] 11.2 Add a unit test that constructs the app with `SIRMAAI_GATEWAY_ENABLED=true` but empty `SIRMAAI_FERNET_KEY` and asserts startup raises `RuntimeError("SIRMAAI_FERNET_KEY required when SIRMAAI_GATEWAY_ENABLED=true")`. (Catch via `pytest.raises` over the lifespan context manager.)

- [ ] **Task 12: Quality gates + sprint-status update (AC: 13, 15)**
  - [ ] 12.1 From `eusolicit-app/`: `make lint` — must be clean. Auto-fix import sort with `make lint-fix` if needed.
  - [ ] 12.2 `make type-check` — no new mypy errors in `sirmaai_gateway` or `client_api`. Pre-existing errors documented in S04.20's "Known Deviation" carried over unchanged.
  - [ ] 12.3 `make reset-db && make migrate-all` — both new migrations apply cleanly. Validate by `psql -c "\d client.sirmaai_projects"` and `\d gateway.workflow_runs`.
  - [ ] 12.4 `make test-service SVC=client-api` and `make test-service SVC=sirmaai-gateway` — both clean. New tests pass; existing tests unaffected.
  - [ ] 12.5 `make test-integration` — new schema-isolation tests pass; new project-cache integration tests pass.
  - [ ] 12.6 `make coverage` — sirmaai-gateway coverage should be flat-or-better vs. S04.20 baseline (~83.36%); the new `project_cache.py`, `cache_invalidation_consumer.py`, `exceptions.py`, model files all contribute test-covered surface.
  - [ ] 12.7 Update `eusolicit-docs/implementation-artifacts/sprint-status.yaml`: flip `4-21-sirmaai-schema-migrations-and-mapping-cache: backlog` → `ready-for-dev` (this is handled by `bmad-create-story` automatically; dev-story flips to `in-progress` on pickup; code-review flips to `done`). Surgical edit only — preserve all comments and structure per `project_sprint_status_file.md` memory.

## Dev Notes

### Architecture: How This Story Fits

Per the **2026-05-12 SirmaAI Amendment** to Epic 4 (`epics/E04-ai-gateway-service.md` lines 500–530, `S04.21` row) and **architecture amendment §4.1** (`architecture-amendment-2026-05-12-sirmaai.md` lines 333–438), the gateway pivots from being a stateless KraftData proxy with an in-process YAML agent registry to being a tenant-aware SirmaAI broker with persistent state in `client.sirmaai_projects` (tenant↔Project map + per-Project encrypted api-key + agent_map) and `gateway.workflow_runs` / `gateway.webhook_subscriptions`. This story is the **foundation** for that pivot:

```
S04.20 (renamed surface, flag) ──┐
                                  │
                                  ▼
                            S04.21 (THIS STORY: schema + cache foundation)
                                  │
              ┌───────────────────┼─────────────────────────────┐
              ▼                   ▼                             ▼
           S04.22              S04.23                        S04.24-26
        (Fernet vault       (logical-name              (async-run, webhooks,
         + rotation;         resolution via             reconciler — all
         publishes the       agent_map)                 write workflow_runs)
         sirmaai.key_rotated
         event that THIS
         story's consumer
         invalidates on)
```

**Why two migrations not one.** Schema isolation (ADR-001) requires each schema's DDL to live in its owning service's alembic chain. `client.sirmaai_projects` is owned by `client-api`'s `client` schema; `gateway.webhook_subscriptions` + `gateway.workflow_runs` are owned by `sirmaai-gateway`'s `gateway` schema. Combining them into a single migration in one service would force that service's alembic chain to do DDL on a foreign schema — which the service role can't do at runtime (only `migration_role` can). Splitting along schema lines keeps each service's chain self-contained.

**Why the dependency order matters.** `make migrate-all` runs `client-api` before `sirmaai-gateway` (per `Makefile` `MIGRATION_ORDER` line 181). This is essential: `gateway.workflow_runs.company_id REFERENCES client.companies(id)` requires `client.companies` to exist when the FK is declared. The order is already correct in the project — no Makefile change needed.

### Pre-Pivot Schema State — Files / Tables You Must Understand Before You Start

**Existing gateway-schema tables (S04.01–S04.10 era, preserved verbatim):**

```
gateway.agent_executions       (S04.08; from sirmaai-gateway alembic 003_agent_executions)
gateway.webhook_log            (S04.07; from sirmaai-gateway alembic 002_webhook_log)
```

These two tables remain in service for the original `ai-gateway` code path while `SIRMAAI_GATEWAY_ENABLED=false`. The new tables (`webhook_subscriptions`, `workflow_runs`) are net-additions — no rename, no drop, no data migration.

**Existing client-schema tables you must NOT modify:** all 71 migrations through `071_create_nps_responses.py` are append-only history. Migration `072` adds one table + grants and nothing else.

**Existing init-script grants you must NOT modify:**

`eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql` (lines 123–134 for the `ai_gateway_role` / `gateway` schema block; lines 213, 222, 231, 240, 248, 265 for `migration_role` grants). The `migration_role SUPERUSER` regression flagged in the S04.20 review (B-1) MUST NOT be re-introduced — verify your working tree's init script is at the original `LOGIN PASSWORD 'migration_password'` only.

### `client.sirmaai_projects` ORM Model — Exact Shape

```python
# services/client-api/src/client_api/models/sirmaai_project.py
from __future__ import annotations

import uuid
from datetime import datetime
from uuid import uuid4

import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from .base import Base


class SirmaAIProject(Base):
    """Tenant ↔ SirmaAI Project mapping with encrypted per-Project api-key.

    One row per EU Solicit company. The api_key_encrypted column holds
    Fernet ciphertext (Epic 9 canonical) for the SirmaAI Project bearer
    token used on outbound calls. agent_map is the per-tenant adapter
    layer (logical_name → SirmaAI agent UUID) introduced when the
    agents.yaml registry was retired in the 2026-05-12 amendment.

    See: architecture-amendment-2026-05-12-sirmaai.md §4.1 (lines 342-358),
         ADR-018 (SirmaAI as agentic substrate, Topology A).
    """

    __tablename__ = "sirmaai_projects"
    __table_args__ = (
        sa.CheckConstraint(
            "provisioning_status IN ('pending','provisioned','failed','archived')",
            name="check_sirmaai_projects_status_valid",
        ),
        {"schema": "client"},
    )

    id: Mapped[uuid.UUID] = mapped_column(
        sa.UUID(as_uuid=True),
        primary_key=True,
        default=uuid4,
        server_default=sa.text("gen_random_uuid()"),
    )
    company_id: Mapped[uuid.UUID] = mapped_column(
        sa.UUID(as_uuid=True),
        sa.ForeignKey("client.companies.id", ondelete="CASCADE"),
        nullable=False,
        unique=True,  # one Project per company
    )
    sirmaai_org_id: Mapped[str] = mapped_column(sa.Text, nullable=False)
    sirmaai_project_id: Mapped[str] = mapped_column(sa.Text, nullable=False, unique=True)
    api_key_encrypted: Mapped[bytes] = mapped_column(sa.LargeBinary, nullable=False)
    api_key_rotated_at: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), nullable=False, server_default=sa.text("now()")
    )
    n8n_subdomain: Mapped[str | None] = mapped_column(sa.Text, nullable=True)
    agent_map: Mapped[dict] = mapped_column(
        JSONB, nullable=False, server_default=sa.text("'{}'::jsonb")
    )
    provisioning_status: Mapped[str] = mapped_column(
        sa.Text, nullable=False, server_default=sa.text("'pending'")
    )
    provisioning_error: Mapped[str | None] = mapped_column(sa.Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), nullable=False, server_default=sa.text("now()")
    )
    updated_at: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), nullable=False, server_default=sa.text("now()")
    )
    archived_at: Mapped[datetime | None] = mapped_column(sa.DateTime(timezone=True), nullable=True)
```

### `gateway.workflow_runs` ORM Model — Cross-Schema FK Boundary

```python
# services/sirmaai-gateway/src/sirmaai_gateway/models/workflow_run.py
from __future__ import annotations

import uuid
from datetime import datetime
from uuid import uuid4

import sqlalchemy as sa
from sqlalchemy.dialects.postgresql import JSONB
from sqlalchemy.orm import Mapped, mapped_column

from . import Base  # sirmaai_gateway-local declarative Base


class WorkflowRun(Base):
    """One row per SirmaAI agent/team/workflow run.

    The 5-min reconciler (S04.26) scans the partial index
    ix_workflow_runs_nonterminal to converge pending/running rows
    via GET /jobs/{jobId}/status. Webhooks (S04.25) and the reconciler
    are eventually-consistent against the same row — see
    architecture-amendment-2026-05-12-sirmaai.md §4.4 "idempotent
    against the reconciler" guidance.

    NOTE: company_id is a cross-schema FK to client.companies(id) —
    the only sanctioned exception to ADR-001 schema isolation in
    this schema. No ORM relationship() is declared (gateway code
    must not navigate to client objects). See architecture amendment
    line 437.
    """

    __tablename__ = "workflow_runs"
    __table_args__ = (
        sa.CheckConstraint(
            "run_type IN ('agent','team','workflow')",
            name="check_workflow_runs_run_type_valid",
        ),
        sa.CheckConstraint(
            "status IN ('pending','running','completed','failed','cancelled')",
            name="check_workflow_runs_status_valid",
        ),
        sa.CheckConstraint(
            "octet_length(payload_excerpt::text) <= 16384",
            name="check_workflow_runs_payload_excerpt_size",
        ),
        {"schema": "gateway"},
    )

    id: Mapped[uuid.UUID] = mapped_column(
        sa.UUID(as_uuid=True),
        primary_key=True,
        default=uuid4,
        server_default=sa.text("gen_random_uuid()"),
    )
    company_id: Mapped[uuid.UUID] = mapped_column(
        sa.UUID(as_uuid=True),
        sa.ForeignKey("client.companies.id"),  # cross-schema; no ondelete; no relationship()
        nullable=False,
    )
    eusolicit_run_id: Mapped[uuid.UUID] = mapped_column(
        sa.UUID(as_uuid=True), nullable=False, unique=True
    )
    sirmaai_run_id: Mapped[str | None] = mapped_column(sa.Text, nullable=True)
    sirmaai_job_id: Mapped[str | None] = mapped_column(sa.Text, nullable=True)
    run_type: Mapped[str] = mapped_column(sa.Text, nullable=False)
    status: Mapped[str] = mapped_column(
        sa.Text, nullable=False, server_default=sa.text("'pending'")
    )
    started_at: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), nullable=False, server_default=sa.text("now()")
    )
    completed_at: Mapped[datetime | None] = mapped_column(sa.DateTime(timezone=True), nullable=True)
    last_polled_at: Mapped[datetime | None] = mapped_column(sa.DateTime(timezone=True), nullable=True)
    error_message: Mapped[str | None] = mapped_column(sa.Text, nullable=True)
    payload_excerpt: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
```

### `ProjectCache` — Shape and Invariants

```python
# services/sirmaai-gateway/src/sirmaai_gateway/services/project_cache.py
from __future__ import annotations

import asyncio
import base64
import json
import uuid
from typing import TYPE_CHECKING

import structlog
from pydantic import BaseModel
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

from eusolicit_common.crypto import FernetCrypto
from eusolicit_common.observability.metrics import counter  # or whatever helper exists

from sirmaai_gateway.services.exceptions import TenantNotProvisionedError

if TYPE_CHECKING:
    import redis.asyncio

log = structlog.get_logger(__name__)

# Metrics — registered once at module import; safe under re-import in tests.
_HITS = counter("sirmaai_project_cache_hits_total", "...", labelnames=("service",))
_MISSES = counter("sirmaai_project_cache_misses_total", "...", labelnames=("service",))


class ProjectMapping(BaseModel):
    sirmaai_project_id: str
    sirmaai_org_id: str
    api_key_plaintext: str    # decrypted in get() — NEVER persisted, NEVER logged
    agent_map: dict[str, str]
    n8n_subdomain: str | None
    provisioning_status: str


class ProjectCache:
    """Redis-backed cache for client.sirmaai_projects with 5-min TTL.

    Cache key:    sirmaai:project:by-company:{company_id}
    Cache value:  JSON dict with base64'd api_key_encrypted bytes
    Decryption:   on every cache read (Fernet ciphertext cached, plaintext returned)
    Invalidation: explicit via invalidate(); driven by sirmaai.key_rotated event
                  consumer in cache_invalidation_consumer.py
    Negative caching: NONE — missing tenant raises TenantNotProvisionedError
                      and does not poison the cache (freshly-provisioned tenants
                      must be visible immediately after the E24 INSERT lands).
    """

    KEY_PREFIX = "sirmaai:project:by-company:"

    def __init__(
        self,
        redis_client: "redis.asyncio.Redis",
        session_factory: async_sessionmaker[AsyncSession],
        crypto: FernetCrypto,
        ttl_seconds: int = 300,
        service_name: str = "sirmaai-gateway",
    ) -> None:
        self._redis = redis_client
        self._session_factory = session_factory
        self._crypto = crypto
        self._ttl = ttl_seconds
        self._service = service_name

    async def get(self, company_id: uuid.UUID) -> ProjectMapping:
        key = f"{self.KEY_PREFIX}{company_id}"
        cached = await self._redis.get(key)
        if cached is not None:
            _HITS.labels(service=self._service).inc()
            return self._decode_and_decrypt(cached)
        _MISSES.labels(service=self._service).inc()
        async with self._session_factory() as session:
            # 5s query timeout — fail fast.
            row = await asyncio.wait_for(self._fetch_row(session, company_id), timeout=5.0)
        if row is None:
            raise TenantNotProvisionedError(company_id)
        # Cache the ciphertext (not plaintext) + TTL.
        await self._redis.setex(key, self._ttl, self._encode_row(row))
        return self._row_to_mapping(row)

    async def invalidate(self, company_id: uuid.UUID) -> None:
        await self._redis.delete(f"{self.KEY_PREFIX}{company_id}")

    # ... _fetch_row, _decode_and_decrypt, _row_to_mapping, _encode_row helpers
```

### Cache-Invalidation Consumer — Wire-Up

The consumer runs **only** when `SIRMAAI_GATEWAY_ENABLED=true`. It uses the existing `eusolicit_common.events.EventConsumer` abstraction — do not roll your own `xreadgroup` loop. The `STREAMS` and `CONSUMER_GROUPS` dicts in `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py` are the registry — add the new entries there and `bootstrap_event_bus()` handles `XGROUP CREATE … MKSTREAM` idempotently on every service startup.

S04.22 (the next story) will publish to the same stream with this payload:
```json
{
  "event_type": "sirmaai.key_rotated",
  "payload": {"company_id": "<uuid>", "rotated_at": "<iso8601>"},
  "source_service": "sirmaai-gateway",
  "tenant_id": "<company_id>",
  "correlation_id": "<uuid>"
}
```

This story's consumer parses `payload.company_id` (the inner JSON field, after `json.loads(envelope["payload"])`), casts via `uuid.UUID(...)`, and calls `cache.invalidate(company_id)`. Malformed events: log at WARN and ACK — never wedge the stream on a poison message.

### Why the Cache Stores Ciphertext, Not Plaintext

Caching plaintext api-keys in Redis would weaken the at-rest encryption guarantee from `api_key_encrypted BYTEA`. Two attackers' viewpoints:

- **DB compromise alone**: Fernet ciphertext is opaque without the `SIRMAAI_FERNET_KEY`. Safe.
- **Redis compromise alone**: if we cached plaintext, an attacker with `KEYS sirmaai:project:by-company:*` + `GET` would dump every tenant's bearer token. By caching ciphertext, the same compromise yields only ciphertext — the attacker still needs `SIRMAAI_FERNET_KEY` (which is in env vars / Helm secrets, separate compromise surface).
- **Combined compromise (DB + Redis)**: not improved by storing ciphertext in Redis — both surfaces already leak ciphertext only.

Net: storing ciphertext costs one extra Fernet decryption per cache read (~microseconds) and improves the defense-in-depth posture meaningfully. Caching ciphertext is the right call.

### Why Negative Caching Is OFF

The amendment AC says "cache-hit rate ≥99% in steady state". Caching the negative result (`TenantNotProvisionedError` for an unknown `company_id`) would boost the apparent hit rate but would also mean:

- E24 provisions a new company at `t=0` (INSERTs the row in `client.sirmaai_projects`).
- A consumer call to `cache.get(new_company.id)` at `t=10s` returns the cached negative result, raising `TenantNotProvisionedError` — even though the row is present.
- The mistake persists for up to `TTL=300s` until the negative cache entry expires.

This is unacceptable for the onboarding UX (E24). The correct trade-off: cache positive results only; pay the DB lookup cost on misses (rare in steady state because provisioned companies stay provisioned). The TenantNotProvisionedError handler upstream (in S04.23) returns a 503 "Tenant not provisioned yet — retry in a moment" with `Retry-After: 5` rather than blowing up the request stack.

### What Could Go Wrong (Anti-Patterns Catalog)

| Anti-pattern | Why it's wrong | What to do instead |
|---|---|---|
| Add `relationship()` from `WorkflowRun` to `Company` for "convenience" | Violates ADR-001 schema isolation — gateway code shouldn't navigate to client objects | Use `company_id` UUID only; if you need company data, call the cache or `client-api` |
| Cache plaintext api-keys in Redis to skip the Fernet decrypt | Weakens at-rest encryption (see "Why the Cache Stores Ciphertext" above) | Always cache ciphertext; decrypt on every cache read |
| Use `==` to compare `X-SirmaAI-Signature` HMAC | Timing attack vector — project rule + E04 retrospective lesson | `hmac.compare_digest()` (this story doesn't do HMAC, but S04.25 will — establishing the import paths here helps) |
| Skip `from __future__ import annotations` at the top of new modules | Project rule violation; breaks future PEP 563 stringification | Always include as line 1 of every new `.py` |
| Use `print` / stdlib `logging` instead of `structlog` | Project rule violation; loses structured fields | `log = structlog.get_logger(__name__)` everywhere |
| `await session.commit()` inside a test | Breaks `db_session` rollback isolation; corrupts parallel tests | Never commit in tests; the fixture rolls back |
| Forget to update `models/__init__.py` | `Base.metadata` doesn't know about the new ORM class → `alembic autogenerate` thinks the table is unmanaged | Import + export every new model class in `__init__.py` |
| Cache on Redis DB 0 in tests (instead of DB 1) | Project test isolation rule: app uses DB 0, tests use DB 1 (`clean_redis` flushes DB 1 only) | Tests use the `clean_redis` fixture; production uses DB 0 — never mix |
| Use bare `except:` to swallow event-consumer errors | Project rule violation; hides actual failures | Catch `json.JSONDecodeError`, `ValueError`, `KeyError` specifically; log + ACK + continue |
| Cache invalidation consumer starts even when flag is `False` | Breaks the S04.20 invariant that pre-amendment paths are bit-identical | Gate every cache-related task on `if settings.sirmaai_gateway_enabled:` |
| Combine `client.sirmaai_projects` DDL with `gateway.*` DDL in one migration | Schema-isolation violation; `migration_role` can do it but the runtime services then both think they own the schema | Two migrations: 072 in client-api, 004 in sirmaai-gateway |
| Forget the GRANT block — gateway can't SELECT `client.sirmaai_projects` at runtime | Cache misses raise `permission denied`, not `TenantNotProvisionedError` | Add the GRANT to migration 072; verify in `test_db_schema_isolation.py` |
| Cache negative results (TenantNotProvisionedError) | New-tenant onboarding UX breaks for up to 5 minutes | Cache positive only; pay the rare DB miss on unknowns |

### Previous Story Intelligence (from S04.20)

- **Service is renamed; the Python package is `sirmaai_gateway` (not `ai_gateway`)** — all imports use `sirmaai_gateway.*`. The settings class is `SirmaAIGatewaySettings`. The `service_name` default is `"sirmaai-gateway"`.
- **The `SIRMAAI_GATEWAY_ENABLED` flag exists on `SirmaAIGatewaySettings`** with default `False`. All new SirmaAI code paths in this story MUST gate on it; the pre-amendment KraftData proxy path stays the runtime behaviour when the flag is `False`. See S04.20 AC #13 + Dev Notes (file `4-20-service-rename-and-flag-scaffold.md` lines 158–202).
- **Docker-compose network alias `ai-gateway` is still in place** (S04.20 AC #6); existing consumers resolve `http://ai-gateway:8004` via the alias. This story doesn't change that — no consumer-facing surface is modified.
- **`ai_gateway_role` is the DB role name and is intentionally NOT renamed** (S04.20 AC #14a). The role name is DB-internal; renaming would be a destructive op. Migrations and `GRANT` statements in this story reference `ai_gateway_role` (NOT `sirmaai_gateway_role`).
- **The S04.20 review identified a security regression** (`migration_role` granted `SUPERUSER` in working tree). Before starting work, run `git diff infra/postgres/init/01-init-schemas-and-roles.sql` and confirm it's clean. If `SUPERUSER` is present, do NOT proceed — revert the file first.
- **The S04.20 review also flagged contaminated working tree** with out-of-scope changes from other stories. Before committing, run `git status` and stage **only** the files in this story's File List. Use explicit `git add <path>` per file, never `git add -A`.
- **Coverage gate was 83.36% in S04.20** (pre-existing gap in `routers/health.py` `/ready` probe). This story should hold or improve that — `make coverage` after Task 12 must not regress.

### Pre-Pivot Code You Must Read Before You Start

Read each file completely before touching its sibling new file. This is the project's #1 cause of review cycles (per `feedback_investigate_dont_quiz.md` memory + Step-3 directive in the workflow).

| File to read | Why |
|---|---|
| `eusolicit-app/services/client-api/alembic/versions/054_create_crm_stage_mappings.py` | Closest peer migration pattern (CHECK constraint, FK to client schema table, UNIQUE constraint, partial index). Modelled on this. |
| `eusolicit-app/services/client-api/src/client_api/models/crm_connection.py` | Closest peer ORM model — Fernet-encrypted column, CHECK constraints in `__table_args__`, `{"schema": "client"}` placement, `Mapped[]` typing. |
| `eusolicit-app/services/client-api/src/client_api/models/__init__.py` | Where to add the `SirmaAIProject` export. Note the alphabetical-by-table-name convention. |
| `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/models/__init__.py` | Where to add `WebhookSubscription` and `WorkflowRun` exports. Existing pattern uses the `Base` from `base.py` (or `__init__.py` depending on layout — check). |
| `eusolicit-app/services/sirmaai-gateway/alembic/versions/003_agent_executions.py` | The migration right before yours. Read it to see the gateway-schema convention. `down_revision = "003"` chains your new 004 on top. |
| `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/crypto.py` | `FernetCrypto` constructor signature, encrypt/decrypt method shapes, `MultiFernet` support for rotation overlap. |
| `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/publisher.py` | Envelope shape for `EventPublisher.publish()` — your tests publish via this; your consumer parses this shape. |
| `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py` | The `STREAMS` and `CONSUMER_GROUPS` registries. Add your two entries here. |
| `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/consumer.py` | `EventConsumer` abstraction — how `xreadgroup` is wrapped and how messages are ACKed. |
| `eusolicit-app/services/notification/src/notification/core/token_crypto.py` | A working `Fernet` integration in another service. Establishes the "fail fast on missing key" pattern your `RuntimeError` mirrors. |
| `eusolicit-app/tests/integration/test_db_schema_isolation.py` | Existing schema-isolation test pattern. Extend this rather than fork it. Lines 45–62 show the role-schema map; lines 65–100 show the asyncpg helpers. |
| `eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql` lines 117–134, 213–248 | The grant blocks you're extending. **Do not modify this file directly** — your migration adds GRANTs via `op.execute()` so they ship with alembic, not with the init script. (Reset DB + migrate-all is the source of truth in dev; in prod, alembic is the only path.) |
| `eusolicit-docs/implementation-artifacts/4-20-service-rename-and-flag-scaffold.md` | The previous story. Sections "Existing Things You MUST Preserve Unchanged" and "Senior Developer Review" carry hard lessons that apply here. |

### Test Standards (project `CLAUDE.md` + `eusolicit-app/` delivery instructions)

- Python 3.12, `from __future__ import annotations` at the top of every new module.
- `@pytest.mark.unit` — pure logic, no I/O. Model CHECK-constraint tests are `@pytest.mark.integration` because they hit the DB.
- `@pytest.mark.integration` — needs postgres + redis. Use `make infra` to start them, `db_session` + `clean_redis` fixtures from root `tests/conftest.py`.
- Test isolation gold standard:
  - DB: per-test transaction rollback via `db_session` fixture. **Never commit.**
  - Redis: `clean_redis` fixture flushes DB 1; app uses DB 0; never mix.
  - Service tests override `get_db_session` / `get_redis_client` dependencies on the FastAPI app, then clear `app.dependency_overrides` in `finally`.
- `structlog` only — no `print`, no stdlib `logging`.
- ruff line-length 120; rules `I E W F UP`. After landing new files, `make lint-fix` to auto-sort imports.
- Cross-tenant negative test required for any tenant-scoped surface (project delivery rule). The "missing tenant" test in AC #7 covers this for the cache; the schema-isolation test in AC #8 covers it at the DB-role boundary.
- HMAC compare via `hmac.compare_digest()` (not in this story, but the import path will exist in S04.25; don't pre-add it here).

### Test Expectations — Where This Story Lands vs. the Epic-Level Test Design

The epic-level `test-design-epic-04.md` was authored for the **pre-amendment** Epic 4 scope (S04.01–S04.10). It enumerates 55 tests across P0–P3 priorities; none of them cover the SirmaAI amendment work. **No epic-level test design has been authored for the amendment yet** (it's an open backlog item; if TEA produces one for S04.21–S04.31 mid-stream, this story should be updated to cross-reference).

In the absence of an amendment-level test design, the test coverage for this story is derived directly from the AC list:

| Risk / concern | Mitigating test (from this story's AC) |
|---|---|
| Cross-schema FK forgotten / wrong | AC #2 + AC #9 (`test_workflow_run_model.py` cross-schema FK enforcement) |
| Partial index missing → reconciler full-scan in S04.26 | AC #1 + AC #2 (DDL inspection by `make migrate-all` + `\d gateway.workflow_runs`) |
| api-key plaintext leaked into Redis | AC #4 + AC #7 (`test_api_key_decrypted_on_cache_read` peeks the raw Redis value) |
| Negative caching wedges new-tenant onboarding | AC #4 + AC #7 (`test_missing_tenant_raises_tenant_not_provisioned_error` — two calls, both hit DB) |
| Event consumer doesn't invalidate cache → stale credentials after rotation | AC #5 + AC #7 (`test_invalidate_on_key_rotated_event`) |
| Schema-isolation violation (CRUD on `client.*` from gateway role) | AC #6 + AC #8 (`test_ai_gateway_role_has_select_only_on_sirmaai_projects`) |
| Feature flag bypassed → SirmaAI code path runs at `false` | AC #11 (`test_lifespan_flag_gating.py`) |
| Cache-hit rate not observable | AC #10 (Prometheus counter increment unit test) |

**No new high-priority risks introduced by this story.** The amendment-era high-priority risks (signature bypass — S04.25; SSE fragility — salvaged from S04.05; Redis publish gap — salvaged from S04.07) are not in this story's scope.

### Definition of Done (per `eusolicit-app/CLAUDE.md` + delivery instructions)

- [ ] `make lint` clean (ruff)
- [ ] `make type-check` clean (mypy) — no new errors in `sirmaai_gateway` or `client_api`
- [ ] `make reset-db && make migrate-all` succeeds — both migrations apply cleanly in dependency order
- [ ] `make test-service SVC=client-api` passes; coverage on `client_api/models/` ≥ baseline
- [ ] `make test-service SVC=sirmaai-gateway` passes; coverage stays flat or improves vs. S04.20 baseline (83.36%)
- [ ] `make test-integration` passes; new schema-isolation tests pass; new project-cache integration tests pass
- [ ] `make coverage` HTML report inspected; no regression
- [ ] `psql` inspection: `\d client.sirmaai_projects`, `\d gateway.webhook_subscriptions`, `\d gateway.workflow_runs` show the expected columns + indexes + CHECK constraints
- [ ] `psql` inspection: `\dp client.sirmaai_projects` shows `ai_gateway_role=r/migration_role` (SELECT-only)
- [ ] `git status` clean of out-of-scope files before committing — explicit `git add` per file
- [ ] sprint-status.yaml row `4-21-…` is `ready-for-dev` after create-story; `dev-story` flips to `in-progress`; `code-review` flips to `done` — no other rows modified
- [ ] No new CRITICAL or HIGH ruff / mypy warnings; pre-existing warnings from S04.20 documented baseline carried over unchanged

### Cutover & Rollback

This story is **additive**: net-new tables, net-new ORM models, net-new cache module, net-new event consumer, net-new GRANTs. No existing schema mutated; no existing code path changed except the feature-flag-gated lifespan startup.

**Rollback path 1 (mid-story revert)**: `git revert` the merge commit + `cd services/client-api && alembic downgrade -1 && cd ../sirmaai-gateway && alembic downgrade -1`. The downgrade is intentionally idempotent and does not revoke GRANTs (they're harmless to leave in place — the underlying tables no longer exist after downgrade).

**Rollback path 2 (production flag flip)**: `SIRMAAI_GATEWAY_ENABLED=false` everywhere — the cache and consumer are not started, the tables exist but are empty (no E24 provisioning has run), the gateway behaves exactly as it did before S04.21 merged. Safe to leave in this state indefinitely.

**Forward path**: S04.22 lands the Fernet vault + rotation Celery Beat + publishes the `sirmaai.key_rotated` event. This story's consumer subscribes to that event from the moment the flag flips. S04.23 wires the cache into the request path. E24 inserts the first real row into `client.sirmaai_projects`.

### Schema Migrations — Project Discipline Checklist

Per project delivery instructions ("Migration discipline"):

- [ ] **Rollback strategy documented**: yes — `alembic downgrade -1` on both services. See "Cutover & Rollback".
- [ ] **NOT NULL on populated column?**: no — net-new tables, no backfill required.
- [ ] **FK addition?**: yes — `gateway.workflow_runs.company_id → client.companies(id)`. Constraint added at `CREATE TABLE` time on an empty table; no lock-contention risk; no `NOT VALID / VALIDATE CONSTRAINT` two-step needed.
- [ ] **Long-running rewrite?**: no.
- [ ] **Cross-component sync→event?**: yes — cache invalidation is event-driven (`sirmaai.key_rotated` Redis Stream), not synchronous cross-service HTTP. Aligns with project reflex "prefer event over sync cross-component call".
- [ ] **Security-relevant unclear decision?**: no — pattern is established (Fernet via Epic 9 module; ADR-001 schema isolation; `hmac.compare_digest()` reserved for S04.25).

### References

- [Epic 4 amendment, S04.21 row + Cutover plan](../../planning-artifacts/epics/E04-ai-gateway-service.md#2026-05-12-amendment--sirmaai-gateway-refactor) (epic file lines 503, 528–530)
- [Architecture amendment §4.1 — Schema Layout (additions)](../../planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#41-schema-layout-additions) (lines 333–438; reference SQL DDL for `client.sirmaai_projects`, `gateway.webhook_subscriptions`, `gateway.workflow_runs`)
- [Architecture amendment §4.4 — Key Data Patterns (additions)](../../planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#44-key-data-patterns-additions) (lines 441–449; cache-hit ≥99% SLO, per-Project key boundary, reconciler-as-truth)
- [Architecture amendment §5.3 — Event Catalog (additions)](../../planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#53-event-catalog-additions) (lines 466–482; `sirmaai.key_rotated` semantics)
- [ADR-018 — SirmaAI as agentic substrate, Topology A](../../planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#adr-018--sirmaai-as-agentic-substrate-topology-a-singleton-org-project-per-company) (lines 30–60)
- [ADR-001 — Schema isolation invariant](../../planning-artifacts/architecture.md) (project-context lookup; cross-schema FK exception list)
- [Story S04.20 — Service rename + flag scaffold](./4-20-service-rename-and-flag-scaffold.md) (Status field: `service_name`, `SIRMAAI_GATEWAY_ENABLED`, package rename. Senior review findings B-1, H-1, M-3 — apply lessons before committing.)
- [Epic 9 canonical Fernet module](../../planning-artifacts/epics/E09-notifications-alerts-calendar.md) (FernetCrypto in `eusolicit-common`; same module used for OAuth tokens, CRM tokens, soon SirmaAI api-keys)
- [Project memory — `project_sprint_status_file.md`](file:///home/debian/.claude/projects/-home-debian-Projects-eusolicit/memory/project_sprint_status_file.md): sprint-status.yaml is orchestrator-managed; surgical edits only
- [Project memory — `project_test_execution_environment.md`](file:///home/debian/.claude/projects/-home-debian-Projects-eusolicit/memory/project_test_execution_environment.md): host venv lacks service deps; use CI for runtime, host venv only for ruff/collection/syntax
- [Project memory — `project_auto_sync_quality.md`](file:///home/debian/.claude/projects/-home-debian-Projects-eusolicit/memory/project_auto_sync_quality.md): `chore: auto-sync …` commits routinely break builds; explicit `git add` per file
- [Project memory — `feedback_investigate_dont_quiz.md`](file:///home/debian/.claude/projects/-home-debian-Projects-eusolicit/memory/feedback_investigate_dont_quiz.md): for "review and update" tasks, verify from disk first
- [Project `CLAUDE.md`](../../../CLAUDE.md) (service ports, schemas, RBAC, testing strategy, "Critical Patterns")
- [`eusolicit-test-utils` factories](../../../eusolicit-app/packages/eusolicit-test-utils/) (`UserFactory`, `CompanyFactory`, `register_and_verify_with_role`, `create_company_pair` — for the integration tests)

## Dev Agent Record

### Agent Model Used

{{agent_model_name_version}}

### Debug Log References

### Completion Notes List

### File List
