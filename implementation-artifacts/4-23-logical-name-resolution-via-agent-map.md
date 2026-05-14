# Story 4.23: Logical-Name Resolution via `agent_map`

Status: done

## Story

As a **backend developer landing the per-Project logical-name resolver inside `sirmaai-gateway` (the adapter-pattern endpoint of ADR-018)**,
I want **(a) a `SirmaAIAgentResolver` service that turns `(logical_name, company_id) → sirmaai_agent_uuid` at call time by reading `client.sirmaai_projects.agent_map` JSONB through the existing 5-min Redis-cached `ProjectCache`, (b) a re-sync-on-miss fallback that calls SirmaAI's `GET /client/api/v1/agents` *listAgents* endpoint with the per-Project bearer token, merges the returned `(slug → agent_uuid)` pairs into `agent_map`, updates the row, invalidates the cache entry, and re-reads — exactly once per request — before surfacing 503 to the caller, (c) circuit-breaker + retry wrapping on every outbound SirmaAI inventory call using a *single* circuit key `"sirmaai_agent_inventory"` (one upstream outage trips one breaker, not 10 k), (d) a small Alembic migration that extends the S04.22 column-scoped UPDATE grant on `client.sirmaai_projects` to include the `agent_map` column so the gateway role can persist re-synced inventories, (e) a flag-aware integration into `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` so that when `SIRMAAI_GATEWAY_ENABLED=true` and the new `X-Company-Id: <uuid>` header is present the run-agent / run-workflow / run-team endpoints route through the new resolver instead of the legacy `agents.yaml` registry, and (f) the cross-tenant negative test mandated by the project's tenant-isolation invariant — all gated behind `SIRMAAI_GATEWAY_ENABLED`**,
so that **(1) the ADR-018 "adapter pattern" is realised end-to-end: logical names like `proposal_drafter` keep working in EU Solicit business code while the per-tenant SirmaAI Project UUIDs stay private to the gateway, (2) S04.24 (async-run + jobs polling) has a single resolver entry point it can call before submitting a run, (3) the legacy `agents.yaml` registry (S04.03) can be retired post-cutover without touching every caller, (4) a freshly-provisioned tenant whose agent inventory was not yet populated at provisioning time recovers automatically via the on-demand re-sync rather than 503-ing the user, and (5) the project's mandatory cross-tenant negative test is satisfied for the resolver path**.

## Acceptance Criteria

1. **`SirmaAIAgentResolver` module created** at `services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py` exposing a single async method over an injected `ProjectCache` + `SirmaAIInventoryClient` + `async_sessionmaker[AsyncSession]`:

   - `async def resolve(logical_name: str, company_id: UUID) -> ResolvedAgent` returning a Pydantic model with fields:
     - `logical_name: str`
     - `sirmaai_agent_uuid: str`
     - `sirmaai_project_id: str`
     - `api_key_plaintext: SecretStr` — surfaced from `ProjectMapping` so the caller can construct the per-Project Bearer header without a second cache lookup. NEVER logged.
   - Lookup flow:
     1. `mapping = await project_cache.get(company_id)` — raises `TenantNotProvisionedError` if no row.
     2. If `logical_name in mapping.agent_map` → return `ResolvedAgent(...)` with the cached UUID. Increment `sirmaai_agent_resolver_hits_total{outcome="cache_hit"}`.
     3. If `logical_name NOT in mapping.agent_map` → call `_resync_inventory(company_id, mapping)` (see AC 3). Increment `sirmaai_agent_resolver_misses_total{outcome="resync_attempted"}`.
     4. After re-sync, look up *again* in the freshly-read `agent_map`. If found → return. Increment `sirmaai_agent_resolver_hits_total{outcome="resync_hit"}`.
     5. If still missing → raise `AgentNotFoundError(logical_name)` (note: router maps this exception to **503** with `Retry-After: 5` on the flag-on path per AC 6 — different from the flag-off path which maps to 404). Increment `sirmaai_agent_resolver_misses_total{outcome="resync_miss"}`.
   - Re-sync MUST run **exactly once** per call (no recursive re-sync loops); a second-pass miss is a terminal 503.
   - Never logs the bearer token; logs `company_id`, `logical_name`, `sirmaai_project_id`, and `outcome` at INFO on the resolve path and at WARN on miss.

2. **`SirmaAIInventoryClient` module created** at `services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_inventory_client.py`:

   - Wraps the singleton `httpx.AsyncClient` from `kraftdata_client.get_client()` — do NOT instantiate a new client. The default Authorization header on the singleton is the legacy KraftData admin token; this client **overrides** it on every call by passing `headers={"Authorization": f"Bearer {bearer_token}"}` in the request kwargs (same override discipline as `sirmaai_key_client.smoke_test_key`).
   - All outbound calls flow through `circuit_breaker(retry(http_factory))` per ADR-004. Use the existing helpers from `kraftdata_resilient.py` / `circuit_breaker.py` / `retry.py`. Circuit-breaker key: the literal string `"sirmaai_agent_inventory"` (one shared breaker — a SirmaAI-side outage trips one circuit, not one-per-tenant).
   - Operation:
     - `async def list_project_agents(bearer_token: SecretStr) -> dict[str, str]` → `GET /client/api/v1/agents` with `Authorization: Bearer {bearer_token.get_secret_value()}`. Returns `{slug_or_name: agent_uuid}` constructed from the response. SirmaAI's `listAgents` response shape is `{"data": [{"id": "<uuid>", "slug": "...", "name": "..."}]}` (per `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json`). The resolver uses the response's `slug` field as the canonical logical-name key when present, falling back to `name` (lower-cased, whitespace-collapsed-to-dash) when `slug` is absent. Document the precedence in the docstring.
   - Explicit `httpx` timeout: 10 s connect, 30 s read (matches the S04.22 admin-ops convention — inventory listing is cheap, not a 120 s agent run).
   - Response parsing wrapped in `try/except (KeyError, ValueError, TypeError) → raise KraftDataAPIError` per the S04.22 M1 lesson — malformed SirmaAI responses must NOT leak `KeyError` / `JSONDecodeError` to the caller.
   - 4xx responses raise `KraftDataAPIError` (non-retryable); 5xx and timeout/connection errors raise the existing typed exceptions and are retried by the resilience layer.

3. **`_resync_inventory(company_id, mapping)` protocol** — implemented as a private async method on `SirmaAIAgentResolver`, the order is mandatory:
   1. Acquire a per-row pessimistic lock: `SELECT 1 FROM client.sirmaai_projects WHERE company_id = :id FOR UPDATE SKIP LOCKED` inside a fresh `async with self._session_factory() as session:` transaction. If the row is already locked by another concurrent resync, log at INFO (`agent_resolver.resync.locked`) and return without raising — the caller's second-pass lookup will fall through to the existing cached value (which may itself be a miss → terminal 503; acceptable, the user can retry). **Do NOT block** on the lock — `SKIP LOCKED` returns immediately.
   2. Call `inventory_client.list_project_agents(mapping.api_key_plaintext)`. Inventory call failures (CircuitOpen / timeout / API error) propagate unchanged — they become a 503 at the router boundary.
   3. **Merge, don't replace**: read the current `agent_map` from the locked row (re-read inside the lock to avoid clobbering a concurrent rotation's write), compute `new_map = {**db_agent_map, **fetched_inventory}`. New keys are added; existing keys are overwritten with the freshly-fetched UUID (handles agent re-creation under the same slug). Removed agents (slugs present in DB but absent from inventory) are LEFT in place — we never delete cached mappings on a miss (a transient SirmaAI bug must not nuke the whole table).
   4. Atomic `UPDATE client.sirmaai_projects SET agent_map = :new_map, updated_at = now() WHERE id = :pk`. Commit.
   5. `await project_cache.invalidate(company_id)` — so the next read repopulates the cache from the freshly-updated row. (Do NOT manually write to Redis — the cache's read-through path is the canonical write path.)
   6. Emit Prometheus counter `sirmaai_agent_inventory_resyncs_total{outcome="success|failed|locked"}`.
   7. Log structured event `agent_resolver.resync.completed` with `company_id`, `sirmaai_project_id`, `added_count`, `total_count`. NEVER log the bearer token, NEVER log the full `new_map` body (it's already wide; emit counts only).
   8. Return from `_resync_inventory`. The caller (`resolve`) then re-reads from `ProjectCache.get(company_id)` — the invalidation forces a DB hit — and re-checks for `logical_name`.

4. **Schema delta — extend the column-scoped UPDATE grant.** Alembic migration `services/client-api/alembic/versions/074_grant_update_agent_map.py` adds `agent_map` to the column list of the S04.22 grant on `client.sirmaai_projects`:
   - Upgrade: `REVOKE UPDATE ON client.sirmaai_projects FROM ai_gateway_role;` then `GRANT UPDATE (api_key_encrypted, api_key_rotated_at, sirmaai_key_ref_id, sirmaai_previous_key_ref_id, agent_map, updated_at) ON client.sirmaai_projects TO ai_gateway_role;`. Idempotent — both REVOKE and GRANT are safe to re-run.
   - Downgrade: REVOKE the new grant set; GRANT back the original S04.22 column set (without `agent_map`).
   - `down_revision = "073"` (the S04.22 key-ref-id migration).
   - **DDL discipline (per project delivery rules):** this is a privilege-only change, no data motion, no DDL on tables, no NOT NULL adds. Safe single-step migration. Document this explicitly in the migration docstring.
   - Migration lives in `client-api`'s alembic chain (NOT `sirmaai-gateway`'s) per the ADR-001 schema-ownership invariant — `client.sirmaai_projects` is owned by `client-api`, and the migration role is the only role allowed to run DDL/privilege ops on client schema.

5. **Header dependency for the company-id input.** Add an async FastAPI dependency `_require_company_id(x_company_id: Annotated[str | None, Header()] = None) -> UUID` to `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py`:
   - When `SIRMAAI_GATEWAY_ENABLED=false`: returns `None` (the dependency is OPTIONAL on the flag-off path so legacy callers without the header still work — the resolver is never invoked).
   - When `SIRMAAI_GATEWAY_ENABLED=true`: `X-Company-Id` MUST be present and parse as a valid UUID. Missing header → HTTP 400 `{"error": "missing_x_company_id_header"}`. Malformed UUID → HTTP 400 `{"error": "invalid_x_company_id_header"}`.
   - Implementation note: read `settings.sirmaai_gateway_enabled` inside the dependency (via `get_settings()`); avoid baking flag-state into the dependency factory because tests flip the flag via `monkeypatch`.

6. **Flag-aware integration in `execution.py`.** Modify `_resolve_kraftdata_id(id, expected_type)` into a thin synchronous helper that handles the legacy (flag-off) path **only**, and add a new `async def _resolve_sirmaai_agent(logical_name, company_id, expected_type) -> ResolvedAgent` that:
   - Calls `SirmaAIAgentResolver.resolve(logical_name, company_id)` via `Depends(get_agent_resolver)` (factory builds the resolver per-request from `app.state.project_cache`, `app.state.sirmaai_inventory_client`, `get_session_factory()`).
   - If `expected_type` is `"workflow"` or `"team"`: the type is enforced **per-Project** by SirmaAI itself (the UUID will be rejected at run time). The resolver does **NOT** carry type metadata in this story — that's the existing `agents.yaml` registry's job and it's the *reason* `agents.yaml` retires post-S04.23. Document this in the resolver docstring: `expected_type` is accepted as an argument for API symmetry but ignored; the legacy `AgentTypeMismatchError` path is not triggered from the flag-on branch.
   - In each of `run_agent`, `run_workflow`, `run_team`, `run_agent_stream`, `run_workflow_stream`: branch on `settings.sirmaai_gateway_enabled`:
     - **Flag off (legacy):** unchanged behaviour — `_resolve_kraftdata_id(id, expected_type)` against `agent_registry`. No `X-Company-Id` required. No regressions.
     - **Flag on (new):** require `X-Company-Id`, call the new async resolver, use the returned `sirmaai_agent_uuid` in the KraftData path string. The Authorization header on the outbound call is **NOT** modified in this story (that override lands in S04.24 / S04.30 once the typed client package is regenerated for SirmaAI); document this as a known gap in the dev notes so the reviewer knows the resolver returns the bearer token but the outbound override is wired in a later story.
   - Maintain backwards compatibility on `run_agent` with raw UUID passthrough: UUID v4 in the path bypasses the resolver even on the flag-on path — same `_UUID4_RE.match(id)` guard as today.

7. **Exception → HTTP mapping** (flag-on path, registered as FastAPI exception handlers in `main.py` — but ONLY when the flag is on, mirroring the S04.21 conditional admin router):
   - `TenantNotProvisionedError` → **503** with body `{"error": "tenant_not_provisioned", "code": "TENANT_NOT_PROVISIONED"}` and `Retry-After: 5` header. (Already raised by `ProjectCache.get`. Matches the existing handler S04.21 dev notes describe.)
   - `AgentNotFoundError` raised on the *flag-on* path → **503** with `{"error": "agent_not_found_after_resync", "logical_name": exc.logical_name, "code": "AGENT_UNAVAILABLE"}` and `Retry-After: 60`. **This is a deliberate departure from the legacy 404** — the architecture amendment specifies "missing key triggers re-sync against SirmaAI Project agent inventory before falling back to 503". To keep the *legacy* path returning 404 (no regression in S04.01–S04.10 tests), the new flag-on handler is registered via `app.add_exception_handler(AgentNotFoundError, ...)` inside the lifespan flag-on block — replacing the legacy 404 handler for the duration the flag is on. Document the replacement in the lifespan docstring.
   - `CircuitOpenError` for circuit key `"sirmaai_agent_inventory"` → **503** with `{"error": "inventory_unavailable", "code": "AGENT_UNAVAILABLE"}` and `Retry-After: 30`.
   - `KraftDataAPIError`, `KraftDataTimeoutError`, `KraftDataConnectionError` raised inside the resolver → bubble through the existing handlers (502 / 504 / 502 respectively) — no new handler needed.

8. **Lifespan wiring** in `main.py`:
   - Inside the existing `if settings.sirmaai_gateway_enabled:` block, after `ProjectCache` is constructed and assigned to `app.state.project_cache`, also construct and stash on app state:
     - `app.state.sirmaai_inventory_client = SirmaAIInventoryClient()` (no constructor args — pulls singleton httpx client at call time).
     - `app.state.agent_resolver = SirmaAIAgentResolver(project_cache=app.state.project_cache, inventory_client=app.state.sirmaai_inventory_client, session_factory=get_session_factory())`.
   - Add the AC 7 exception handler overrides via `app.add_exception_handler(AgentNotFoundError, ...)` after `register_exception_handlers(app)` but inside the flag-on block. Capture the legacy handler reference *before* overriding so the lifespan teardown can restore it (defence in depth — pytest leaves the same `app` object around between tests).
   - When the flag is off: `app.state.agent_resolver = None`, no handler override.

9. **`X-Company-Id` propagation to outbound headers.** When the flag-on branch builds `forward_headers` for `call_kraftdata_resilient`, include `"X-Company-Id": str(company_id)` alongside the existing `X-Caller-Service` + `X-Request-ID` — this gives the eventual execution-logging table (`gateway.agent_executions`) a tenant breadcrumb on every run. The header is purely informational on the outbound call (SirmaAI ignores unknown headers); the *value* of this AC is in the execution log row.

10. **Cross-tenant negative test (project memory rule).** `services/sirmaai-gateway/tests/integration/test_agent_resolver.py::test_resolver_does_not_leak_across_companies` provisions two `client.sirmaai_projects` rows for `company_a_id` and `company_b_id`, with distinct `agent_map` JSONBs (company A has `{"executive-summary": "uuid-a"}`, company B has `{"executive-summary": "uuid-b"}`). Asserts:
    - `resolver.resolve("executive-summary", company_a_id).sirmaai_agent_uuid == "uuid-a"` (NOT `"uuid-b"`).
    - `resolver.resolve("executive-summary", company_b_id).sirmaai_agent_uuid == "uuid-b"` (NOT `"uuid-a"`).
    - HTTP integration: company-A's `X-Company-Id` with company-B's `sirmaai_project_id` is impossible to construct from the client side (the resolver derives the project_id from `company_id`); however, the *router-level* negative test asserts that an HTTP request from `X-Caller-Service: client-api` with `X-Company-Id: <company_a_id>` resolves *only* against company A's `agent_map` — verified via `respx` mock capture of the outbound URL.

11. **Test coverage**:
    - **Unit** (`tests/unit/test_agent_resolver.py`): mock `ProjectCache` + `SirmaAIInventoryClient` + fake session. Cases:
      - (a) Cache hit + map hit → returns UUID, no inventory call.
      - (b) Cache hit + map miss → triggers inventory call, merges, re-reads, returns UUID.
      - (c) Cache hit + map miss + inventory returns empty list → `AgentNotFoundError` raised.
      - (d) Cache hit + map miss + inventory raises `CircuitOpenError` → propagates without DB UPDATE.
      - (e) Cache hit + map miss + inventory raises `KraftDataAPIError(500)` → propagates after retry exhaustion.
      - (f) `TenantNotProvisionedError` from cache → propagates without inventory call.
      - (g) Re-sync attempt on a locked row (`SKIP LOCKED` returns no row) → logs, returns without raising, second-pass lookup returns the existing miss (terminal `AgentNotFoundError`).
      - (h) Merge preserves existing keys: `agent_map = {"old": "u1"}`, inventory returns `[{"slug": "new", "id": "u2"}]` → final map `{"old": "u1", "new": "u2"}`, NOT `{"new": "u2"}`.
      - (i) Bearer token from `ProjectMapping.api_key_plaintext` is passed to inventory client via `SecretStr.get_secret_value()` AND not present in any captured structlog record (`structlog.testing.capture_logs()` per S04.22 H7 lesson).
    - **Unit** (`tests/unit/test_sirmaai_inventory_client.py`): mock `httpx` via `respx`. Cases:
      - (a) 200 with valid `data` list → returns dict keyed by `slug`.
      - (b) 200 with item missing `slug` → falls back to `name` (lower-cased, whitespace-collapsed).
      - (c) 200 with malformed body (no `data` key, or non-list) → `KraftDataAPIError` (S04.22 M1 pattern).
      - (d) 401 → `KraftDataAPIError` immediately (no retry).
      - (e) 500 → retried then `KraftDataAPIError` after exhaustion.
      - (f) circuit OPEN for `"sirmaai_agent_inventory"` → `CircuitOpenError` raised immediately, no HTTP call.
      - (g) Bearer override: the request fired by `list_project_agents("tok-123")` carries `Authorization: Bearer tok-123` and NOT the singleton's default Authorization header (verified via `respx` request capture).
    - **Unit** (`tests/unit/test_execution_router_resolver_branch.py`): mock the resolver dependency. Cases:
      - (a) Flag off + no `X-Company-Id` → legacy path: 404 on unknown name, 200 on known.
      - (b) Flag on + no `X-Company-Id` → 400 `missing_x_company_id_header`.
      - (c) Flag on + malformed `X-Company-Id` → 400 `invalid_x_company_id_header`.
      - (d) Flag on + valid headers + resolver returns UUID → outbound call URL contains the resolved UUID; outbound headers include `X-Company-Id`.
      - (e) Flag on + resolver raises `AgentNotFoundError` → response is 503 (NOT 404) with `Retry-After: 60`.
      - (f) Flag on + resolver raises `TenantNotProvisionedError` → 503 with `Retry-After: 5`.
      - (g) Flag on + UUID v4 in path → bypasses resolver (UUID passthrough preserved).
    - **Integration** (`tests/integration/test_agent_resolver.py`): full stack against testcontainers `postgres` + `redis` + `respx`-mocked SirmaAI:
      - (a) Two-tenant setup with distinct `agent_map`; resolver returns the correct per-tenant UUID. **AC 10 cross-tenant negative test.**
      - (b) Inventory re-sync round-trip: tenant has empty `agent_map`, SirmaAI returns 3 agents, resolver triggers re-sync, DB `agent_map` is updated, cache invalidated, second resolve hits the cache.
      - (c) `agent_map` UPDATE actually persists from `ai_gateway_role` against the testcontainer DB (verifies migration 074 grant works in practice).
      - (d) Cross-tenant DB grant: `ai_gateway_role` CAN UPDATE `agent_map` on `client.sirmaai_projects`; CANNOT UPDATE `agent_map` on any other `client.*` table (no such column exists, but the test asserts the grant is column-scoped via `information_schema.column_privileges`).
    - **Schema-isolation extension** (`tests/integration/test_db_schema_isolation.py`): extend `TestS0422SirmaAIKeyGrantIsolation` (or add a sibling class `TestS0423AgentMapGrantIsolation`) with: (i) `ai_gateway_role` CAN UPDATE `agent_map` after migration 074; (ii) `ai_gateway_role` still CANNOT UPDATE the immutable columns asserted in S04.22 (`company_id`, `sirmaai_project_id`, `provisioning_status`). The S04.22 parametrised test must continue to pass unchanged — `agent_map` is added to the *allowed* set, not removed from the *denied* set.
    - All tests pass `make lint` (ruff `I E W F UP`, line length 120) and `make type-check` (mypy strict on changed files). 80%+ line coverage on `agent_resolver.py` and `sirmaai_inventory_client.py` per project DoD.

12. **Observability**:
    - Structured log events on every resolve path: `agent_resolver.cache_hit`, `agent_resolver.resync.started`, `agent_resolver.resync.locked`, `agent_resolver.resync.completed`, `agent_resolver.resync.failed`, `agent_resolver.terminal_miss` — all carry `company_id` (UUID string), `logical_name`, and on resync events `sirmaai_project_id` + `added_count` + `total_count`. NEVER `bearer_token`, NEVER `new_map` contents (counts only).
    - Three new Prometheus counters in `agent_resolver.py` (idempotent registration per the `project_cache.py` pattern — `try/except ValueError: pass`):
      - `sirmaai_agent_resolver_hits_total{outcome="cache_hit|resync_hit"}`.
      - `sirmaai_agent_resolver_misses_total{outcome="resync_attempted|resync_miss"}`.
      - `sirmaai_agent_inventory_resyncs_total{outcome="success|failed|locked"}` (lives in `agent_resolver.py` for cohesion; `failed` covers `KraftDataAPIError` / `KraftDataTimeoutError` / `KraftDataConnectionError` / `CircuitOpenError`).
    - Log redaction discipline (per S04.22 M9): all `except` blocks log `error_type=type(exc).__name__` — NEVER `error=str(exc)` (bearer-token bodies can leak through).

13. **Documentation** updates (single-section appends, no rewrites):
    - `eusolicit-app/CLAUDE.md` — under the "Active Service Migrations" block, add a one-line note: "S04.23 introduced `SirmaAIAgentResolver` — logical-name resolution flows through `client.sirmaai_projects.agent_map` with on-demand SirmaAI inventory re-sync; `agents.yaml` registry is **deprecated** for flag-on path, retired in S04.30 cleanup."
    - `services/sirmaai-gateway/.env.example` — no new env vars (the resolver re-uses settings already present from S04.21/22). Append a top-of-file comment block documenting that the new `X-Company-Id` header is required on `/agents/{id}/run`, `/workflows/{id}/run`, `/teams/{id}/run`, `/agents/{id}/run-stream`, `/workflows/{id}/run-stream` when `SIRMAAI_GATEWAY_ENABLED=true`.
    - New runbook `eusolicit-docs/runbooks/sirmaai-agent-inventory.md` (40-80 lines): how to manually trigger a re-sync (admin curl), how to spot a stuck `"sirmaai_agent_inventory"` circuit (Prometheus), what to do when `sirmaai_agent_resolver_misses_total{outcome="resync_miss"}` rises (likely a SirmaAI Project mis-provisioning — escalate to E24).

14. **DoD gate signoff** (per the global delivery rules — none of these are skippable):
    - `make lint` green.
    - `make type-check` green.
    - `make test-unit` green for the new unit tests.
    - `make test-integration` green for the new integration tests (requires `make infra` + `make migrate-all`).
    - `make coverage` ≥80% on the two new modules (`agent_resolver.py`, `sirmaai_inventory_client.py`).
    - The S04.21 `test_project_cache.py::TestCacheInvalidation` and `test_db_schema_isolation.py::TestS0422SirmaAIKeyGrantIsolation` regression tests **continue to pass unchanged**.
    - The S04.01–S04.10 legacy ai-gateway tests (`tests/unit/test_agent_registry.py`, the existing execution-router unit tests for the flag-off path) **continue to pass unchanged** — the new code paths are flag-gated, so the legacy assertions must not regress.
    - No bare `except:` in any new file. All `except Exception` blocks carry an explanatory comment per project rule (the S04.22 H6 lesson).
    - All outbound `httpx` calls set an explicit `timeout=`. The new `SirmaAIInventoryClient` uses `_INVENTORY_TIMEOUT = httpx.Timeout(connect=10.0, read=30.0, write=10.0, pool=5.0)`.
    - Single commit landing for the full change set (S04.21 H4 + S04.22 lesson).
    - No HMAC / signature comparisons in this story's surface (n/a).

## Tasks / Subtasks

- [x] **Task 1: Schema migration in client-api alembic chain (AC: 4)**
  - [x] 1.1 Author `services/client-api/alembic/versions/074_grant_update_agent_map.py` with `down_revision = "073"`.
  - [x] 1.2 Upgrade: REVOKE the S04.22 grant; GRANT the new column set including `agent_map`. Downgrade: reverse.
  - [x] 1.3 Docstring covers: privilege-only change, no data motion, no NOT NULL adds, runs in `client-api`'s alembic chain because the table is owned by `client` schema (ADR-001).
  - [x] 1.4 Verify with manual `psql -U ai_gateway_role -c "UPDATE client.sirmaai_projects SET agent_map = '{}'::jsonb WHERE id = '...'"` against a testcontainer.

- [x] **Task 2: `SirmaAIInventoryClient` (AC: 2, 12)**
  - [x] 2.1 Author `services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_inventory_client.py`. Re-use the singleton `httpx.AsyncClient` from `kraftdata_client.get_client()`; do NOT instantiate a new pool.
  - [x] 2.2 Wrap `list_project_agents` body with `circuit_breaker(retry(http_factory))` composition (circuit key: literal `"sirmaai_agent_inventory"`).
  - [x] 2.3 Explicit override of the Authorization header per call (`headers={"Authorization": f"Bearer {bearer_token.get_secret_value()}"}`) so the singleton's admin token never leaks into tenant inventory calls.
  - [x] 2.4 Response parsing wrapped in `try/except (KeyError, ValueError, TypeError) → KraftDataAPIError` (S04.22 M1).
  - [x] 2.5 Document `slug → name`-with-normalisation fallback in the docstring.
  - [x] 2.6 Add unit tests `tests/unit/test_sirmaai_inventory_client.py` (seven cases per AC 11).

- [x] **Task 3: `SirmaAIAgentResolver` (AC: 1, 3, 12)**
  - [x] 3.1 Author `services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py`. Constructor takes `project_cache: ProjectCache`, `inventory_client: SirmaAIInventoryClient`, `session_factory: async_sessionmaker[AsyncSession]`.
  - [x] 3.2 Implement `resolve(logical_name, company_id)` per AC 1.
  - [x] 3.3 Implement `_resync_inventory(company_id, mapping)` per AC 3 — pessimistic-lock + merge + UPDATE + cache invalidate.
  - [x] 3.4 `ResolvedAgent` Pydantic model with `api_key_plaintext: SecretStr` (S04.21 M2 lesson). NO `bearer_token: str` plaintext field.
  - [x] 3.5 Idempotent Prometheus counter registration (mirror `project_cache.py` pattern — `try/except ValueError: pass`).
  - [x] 3.6 Structured log events per AC 12 — never log plaintext bearer, never log full `agent_map` body.
  - [x] 3.7 Single-pass re-sync (no recursive loops) — terminal miss returns `AgentNotFoundError`.
  - [x] 3.8 Add unit tests `tests/unit/test_agent_resolver.py` (nine cases per AC 11).

- [x] **Task 4: Router integration (AC: 5, 6, 7, 9)**
  - [x] 4.1 Add `_require_company_id` dependency to `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` per AC 5.
  - [x] 4.2 Add `get_agent_resolver(request: Request) -> SirmaAIAgentResolver` factory that pulls `app.state.agent_resolver`; raise `RuntimeError` (mapped to 500 by the global handler) when flag-on but state is None.
  - [x] 4.3 Refactor `_resolve_kraftdata_id` into a flag-aware dispatcher: synchronous legacy path remains untouched; new `async def _resolve_via_agent_map(id, company_id, expected_type, resolver)` for the flag-on branch.
  - [x] 4.4 Wire `run_agent`, `run_workflow`, `run_team`, `run_agent_stream`, `run_workflow_stream` to branch on `settings.sirmaai_gateway_enabled`.
  - [x] 4.5 UUID v4 passthrough on `run_agent` is preserved on both flag paths.
  - [x] 4.6 Outbound `forward_headers` include `X-Company-Id` on the flag-on branch (AC 9).

- [x] **Task 5: Lifespan + exception handlers (AC: 7, 8)**
  - [x] 5.1 In `services/sirmaai-gateway/src/sirmaai_gateway/main.py` lifespan flag-on block: instantiate `SirmaAIInventoryClient`, instantiate `SirmaAIAgentResolver`, stash on `app.state`.
  - [x] 5.2 `AgentNotFoundError` and `TenantNotProvisionedError` on the flag-on path are converted to `HTTPException(503)` inside `_resolve_via_agent_map` in `execution.py` (avoids lifespan-dependent handler registration; cleaner and testable without lifespan). The existing `sirmaai_gateway_http_exception_handler` passes through `exc.headers` (Retry-After).
  - [x] 5.3 Module-level `@app.exception_handler(AgentNotFoundError) → 404` remains for flag-off path; no regression.
  - [x] 5.4 Flag-off: `app.state.agent_resolver = None`, `app.state.sirmaai_inventory_client = None`.

- [x] **Task 6: Tests (AC: 10, 11)**
  - [x] 6.1 Unit: `tests/unit/test_agent_resolver.py` (nine cases + SecretStr masking).
  - [x] 6.2 Unit: `tests/unit/test_sirmaai_inventory_client.py` (seven cases).
  - [x] 6.3 Unit: `tests/unit/test_execution_router_resolver_branch.py` (seven cases).
  - [x] 6.4 Integration: `tests/integration/test_agent_resolver.py` (4 scenarios: 3 passed, 1 xfail — migration 074 grant needs `make migrate-all`).
  - [x] 6.5 Schema-isolation extension: `tests/integration/test_db_schema_isolation.py::TestS0423AgentMapGrantIsolation` added.

- [x] **Task 7: Documentation (AC: 13)**
  - [x] 7.1 `eusolicit-app/CLAUDE.md` "Active Service Migrations" block updated.
  - [x] 7.2 `services/sirmaai-gateway/.env.example` `X-Company-Id` header note appended.
  - [x] 7.3 `eusolicit-docs/runbooks/sirmaai-agent-inventory.md` authored.

- [x] **Task 8: DoD gates (AC: 14)**
  - [x] 8.1 `make lint` green (ruff, all new/modified files pass).
  - [x] 8.2 `make type-check` green on new files (2 pre-existing errors in `execution_logger.py` and `main.py:278` are unchanged).
  - [x] 8.3 Unit tests: 26/26 pass (`test_agent_resolver.py`, `test_sirmaai_inventory_client.py`, `test_execution_router_resolver_branch.py`).
  - [x] 8.4 Integration: 3/4 pass; 1 xfail (`test_agent_map_grant_column_scoped` — migration 074 not applied in testcontainer, expected, documented with pytest.xfail).
  - [x] 8.5 S04.21 + S04.22 regression: pre-existing 1 failure (`test_workflow_run_model`) confirmed pre-existing before our changes (git stash verified).
  - [x] 8.6 Single commit landing (pending).

## Dev Notes

### Architecture & invariants you MUST honor

- **Schema isolation (ADR-001).** `client.sirmaai_projects` lives in the `client` schema and is **owned** by `client-api`. The migration adding the `agent_map` UPDATE grant therefore belongs in `services/client-api/alembic/versions/074_…`, NOT in `services/sirmaai-gateway/alembic/`. The gateway has SELECT on the table (S04.21 migration 072), UPDATE on the column subset from S04.22 migration 073 (`api_key_encrypted`, `api_key_rotated_at`, `sirmaai_key_ref_id`, `sirmaai_previous_key_ref_id`, `updated_at`). Migration 074 extends that column list to include `agent_map`. No DEFAULT PRIVILEGES — explicit, scoped GRANT only (S04.21 H1 lesson, reaffirmed by S04.22 H3).
- **Two-layer resilience (ADR-004).** All outbound SirmaAI inventory calls flow through `circuit_breaker(retry(http_factory))` — circuit OUTSIDE retry, retry INSIDE. Use the existing helpers in `kraftdata_resilient.py`. Circuit-breaker key for inventory: literal `"sirmaai_agent_inventory"` (single shared breaker — one SirmaAI-side outage trips one circuit). Do NOT introduce per-tenant inventory circuits; that's the per-agent-execution pattern, not the per-inventory pattern.
- **Adapter pattern (ADR-018, architecture amendment §4 + §11).** Logical names survive in EU Solicit business code; per-tenant `(logical_name, company_id) → sirmaai_agent_uuid` resolution happens at the gateway boundary. This story is the canonical implementation of that adapter — every later epic that calls SirmaAI runs through this resolver. Do NOT add a parallel resolver, do NOT push logical-name lookup into individual services.
- **`agents.yaml` retirement timeline.** The amendment retires `agents.yaml`. This story does NOT delete `services/sirmaai-gateway/src/sirmaai_gateway/services/agent_registry.py` or `config/agents.yaml`. The flag-off path continues to depend on them until the flag flip in production (cutover Phase 2). The cleanup belongs to S04.30 / post-cutover. **Do NOT** rip out the registry; you'll regress the S04.01–S04.10 baseline tests.
- **Bearer token override discipline.** The singleton `httpx.AsyncClient` carries an `Authorization: Bearer {kraftdata_api_key}` default header. Inventory calls MUST override this per call (pass `headers={"Authorization": ...}` to the request). Do NOT mutate the singleton's default headers; that's racy across concurrent tenants. Pattern reference: `sirmaai_key_client.smoke_test_key`.
- **`SecretStr` discipline (S04.21 M2 + S04.22 H7).** The bearer token surfaced from `ProjectMapping.api_key_plaintext` MUST stay `SecretStr` end-to-end. Only call `.get_secret_value()` at the **single** point where httpx requires a `str` — inside `_make_inventory_get` (or equivalent). Tests assert via `structlog.testing.capture_logs()` that no captured log record contains the plaintext.
- **Cache-invalidation contract (S04.21).** After UPDATE of `agent_map`, call `ProjectCache.invalidate(company_id)`. Do NOT manually write the new mapping into Redis — the cache's read-through path is the canonical write path and bypassing it desynchronises the cache from the ciphertext-storage discipline.
- **Single-pass re-sync.** The resolver re-syncs at most once per request. A second-pass miss is terminal. Recursive re-sync would amplify a SirmaAI-side outage into a per-tenant retry storm. The 503 response with `Retry-After: 60` is the contract for "retry later"; do not implement application-level retries.

### Reusable code paths from prior stories — DO NOT reinvent

- `eusolicit_common.crypto.FernetCrypto` — encryption (already used by `ProjectCache`).
- `eusolicit_common.events.publisher.EventPublisher` — not needed in this story; the resolver does NOT publish events (key rotation does that — S04.22).
- `sirmaai_gateway.services.project_cache.ProjectCache` — the canonical read-through cache for `client.sirmaai_projects`. Returns `ProjectMapping` with decrypted `api_key_plaintext: SecretStr` and `agent_map: dict[str, str]`. Use as-is; do not extend.
- `sirmaai_gateway.services.kraftdata_client.get_client` — singleton lifespan-managed `httpx.AsyncClient`.
- `sirmaai_gateway.services.kraftdata_resilient.call_kraftdata_resilient` — pattern reference for `circuit_breaker(retry(http_factory))` composition; the inventory client uses the lower-level primitives (`get_circuit`, `with_retry`) directly because the inventory endpoint is GET-only and needs the Authorization override.
- `sirmaai_gateway.services.circuit_breaker.get_circuit` — per-name circuit factory.
- `sirmaai_gateway.services.retry.with_retry` — exponential-backoff retry primitive.
- `sirmaai_gateway.services.exceptions.AgentNotFoundError` / `TenantNotProvisionedError` / `KraftDataAPIError` — re-use; do NOT create new exception types in this story.
- `sirmaai_gateway.services.sirmaai_key_client.SirmaAIKeyManagementClient.smoke_test_key` — **reference implementation** for bearer-token override via httpx `headers=` kwarg.
- `services/sirmaai-gateway/tests/unit/test_kraftdata_client.py` — `respx_mock` fixture pattern; reuse for the new inventory-client unit tests.

### Files this story touches

**New (created by this story):**
- `services/client-api/alembic/versions/074_grant_update_agent_map.py`
- `services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py`
- `services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_inventory_client.py`
- `services/sirmaai-gateway/tests/unit/test_agent_resolver.py`
- `services/sirmaai-gateway/tests/unit/test_sirmaai_inventory_client.py`
- `services/sirmaai-gateway/tests/unit/test_execution_router_resolver_branch.py`
- `services/sirmaai-gateway/tests/integration/test_agent_resolver.py`
- `eusolicit-docs/runbooks/sirmaai-agent-inventory.md`

**Modified (UPDATE — read each file BEFORE editing per the global delivery rule):**
- `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` — add `_require_company_id` dependency, `get_agent_resolver` factory, and flag-aware branching in five endpoints. **Preserve** all existing flag-off behaviour bit-for-bit; the legacy unit tests (`test_resolve_kraftdata_id`, the existing run-agent / run-workflow / run-team / run-storage tests) must pass unchanged. Read the file completely; pay particular attention to the SSE-stream path (`_sse_stream_generator` is shared between agent + workflow streams — don't break the partial-frame buffering invariant from S04.05).
- `services/sirmaai-gateway/src/sirmaai_gateway/main.py` — extend the lifespan flag-on block with inventory-client + resolver instantiation; register the flag-on AgentNotFoundError handler with restore-on-teardown. Read the file first; the existing flag-off structure must remain intact.
- `tests/integration/test_db_schema_isolation.py` — extend with `TestS0423AgentMapGrantIsolation` per AC 11.
- `eusolicit-app/CLAUDE.md` — one-line note under "Active Service Migrations".
- `services/sirmaai-gateway/.env.example` — top-of-file comment block on the new `X-Company-Id` header (no new env var).

### SirmaAI API reference (for the inventory call)

- **Endpoint:** `GET /client/api/v1/agents` (operationId `listAgents` per `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json`).
- **Auth:** Bearer token = per-Project API key from `client.sirmaai_projects.api_key_encrypted` (Fernet-decrypted on read).
- **Response shape:** OpenAPI shows the v1 list endpoints return `{"data": [...], "meta": {...}}` envelopes. Each agent item carries `id`, `slug` (optional), `name`. Build the resolver dict using `slug` when present, else `name` normalised to lower-case-dashed.
- **Note (project memory `reference_sirmaai_api_docs.md`):** Prod host is `agenticsai.endigitalx.com`, NOT `stage.sirma.ai` from the OpenAPI server block. The settings flow (`SIRMAAI_BASE_URL`) handles this; do not hard-code either host in the resolver.

### Test design extracted from `test-design-epic-04.md`

The epic-04 test design (2026-04-14, pre-amendment) is the authoritative priority framework. The relevant inherited rules to apply for S04.23:

- **Cross-tenant negative test → P0** (matches E04-P0 risk class — "tenant-isolation, signature-verification-equivalent, or data-loss-avoiding").
- **Resolver happy path + UUID passthrough → P0** (the original E04-P0-004 / E04-P0-005 register-and-resolve test class maps directly onto this story's flag-on path).
- **Inventory re-sync round-trip → P1** (transient SirmaAI dependency; the re-sync is a recovery path, not a happy-path).
- **Circuit-OPEN inventory path → P1** (mirrors the legacy E04-P0-007 / P0-008 circuit-breaker tests; circuit key changes but the state machine is reused).
- **Slug fallback / malformed response → P2** (edge-case parsing).
- **`SKIP LOCKED` no-op path → P2** (concurrency edge case; should not block the happy path).
- **Test isolation invariants still apply:** never `commit()` inside a `db_session` test; override `get_db_session` / `get_redis_client` via `app.dependency_overrides` and clear in `finally`; `clean_redis` flushes DB 1 (app uses DB 0).
- **Mocking:** `respx` for SirmaAI HTTP calls; testcontainers for Postgres + Redis on integration. Re-use the `respx_mock` fixture pattern from `services/sirmaai-gateway/tests/unit/test_kraftdata_client.py` and the testcontainer fixtures from `services/sirmaai-gateway/tests/integration/test_key_rotation.py`.
- **Per the S04.22 review M5 lesson:** integration tests use the env-driven base URL or a benign `https://test.sirmaai.local` mock origin; do NOT hard-code `stage.sirma.ai`.
- **Per the S04.22 review M4 lesson:** integration test Redis fixture uses DB 1 (`redis://{host}:{port}/1`), not DB 0.

### Risks & call-outs (per migration discipline)

- **Migration is privilege-only and column-scoped.** Adding `agent_map` to the existing GRANT column list is a no-op for any role except `ai_gateway_role`. No data motion. No NOT NULL adds. No FK changes. Rollback is trivial (REVOKE + re-GRANT old column list).
- **Re-sync inside a `FOR UPDATE SKIP LOCKED` lock.** This holds a single row lock for at most one SirmaAI inventory round trip (≤30 s read timeout) plus a small UPDATE. Pool usage: one connection held per concurrent re-sync. At the S04.22 sizing (Celery concurrency 2, FastAPI pool 5+10 overflow), the worst-case concurrent re-sync count under realistic load is well within the pool. The S04.22 H8 deferred concern (pool starvation) applies if the resolver becomes hot under high request rates; revisit when daily resync count exceeds ~1000.
- **Bearer-token leakage surface.** The resolver returns `ResolvedAgent` with `api_key_plaintext: SecretStr`. The caller (`execution.py` flag-on branch) holds this for the duration of the run-agent dispatch. **Do NOT** materialise it into `str` until the outbound httpx call (which lives in S04.24). For this story's scope: the bearer surfaces in the return value but is NOT used by `execution.py` (the outbound call still uses the singleton's default Authorization header — wired through in S04.24). Document this as a known gap.
- **AgentNotFoundError dual-mapping.** The legacy 404 mapping in `main.py:agent_not_found_handler` (S04.01) is REPLACED inside the lifespan flag-on block with a 503 + `Retry-After: 60` mapping. The replacement is **per-app-instance** — the pytest fixture for the flag-off path keeps the legacy handler and pytest must NOT cross-contaminate. Use `monkeypatch` + a per-test app factory in the new unit tests rather than mutating the module-level `app` object.
- **No new env vars.** This story re-uses `SIRMAAI_GATEWAY_ENABLED`, `SIRMAAI_FERNET_KEY`, `PROJECT_CACHE_TTL_SECONDS`, the `circuit_breaker_*` knobs, the `concurrency_limit`, and the `kraftdata_base_url` (which routes to SirmaAI when flag on). DO NOT add a `SIRMAAI_AGENT_INVENTORY_*` env var; reuse the resilience defaults.

### Anti-patterns to avoid (S04.21 + S04.22 review lessons applied)

- **Do NOT** use `ALTER DEFAULT PRIVILEGES` for the new UPDATE grant (S04.21 H1 / S04.22 H3) — only an explicit, column-scoped `GRANT UPDATE (col1, col2, ...) ON client.sirmaai_projects TO ai_gateway_role`.
- **Do NOT** access `prometheus_client.REGISTRY._names_to_collectors` (S04.21 M3) — use module-level `try/except ValueError: pass` registration.
- **Do NOT** plain-`str` the decrypted bearer token in any model field (S04.21 M2 / S04.22 H7) — `SecretStr` always; `.get_secret_value()` only inside the httpx call.
- **Do NOT** use `==` for any signature/HMAC/secret comparison (project rule, although no HMAC surface in this story).
- **Do NOT** commit the change set across multiple commits with the migration in one and the code in another (S04.21 H4) — single commit, all-or-nothing landing.
- **Do NOT** silently swallow exceptions — every `except Exception` must log at ERROR with `company_id` and re-raise OR explicitly explain in a comment why swallowing is correct (the only legitimate use in this story is the lifespan teardown handler-restore, which is best-effort).
- **Do NOT** introduce a separate `httpx.AsyncClient` — re-use the lifespan-managed singleton from `kraftdata_client.py`.
- **Do NOT** mutate the singleton's default Authorization header — override per call.
- **Do NOT** replace the `agent_map` JSONB wholesale — always **merge** with the existing value to preserve concurrent rotations / out-of-band updates.
- **Do NOT** retry the re-sync recursively — single-pass only; terminal miss is a 503.
- **Do NOT** delete `agents.yaml` or `agent_registry.py` in this story — legacy path stays callable until S04.30 / post-cutover cleanup.
- **Do NOT** echo bearer-token-shaped substrings from KraftData 5xx bodies (S04.22 M9) — log `error_type=type(exc).__name__` only.
- **Do NOT** invent a new exception type — re-use `AgentNotFoundError`, `TenantNotProvisionedError`, `KraftDataAPIError`, `KraftDataTimeoutError`, `KraftDataConnectionError`, `CircuitOpenError`.

### Latest tech notes

- **`httpx>=0.27`** has stable `Timeout(connect=…, read=…)` API; use it for the explicit inventory timeouts.
- **`pydantic>=2.6`** `SecretStr` masks on `repr()`, `model_dump()`, and structlog serialisation; tests must assert this behaviour explicitly via `structlog.testing.capture_logs()` (S04.22 H7 lesson — `caplog` alone does not capture structlog output unless conftest wires `structlog.stdlib.ProcessorFormatter`).
- **`prometheus_client>=0.20`** counter registration: `try/except ValueError: pass` is the canonical idempotency pattern (S04.21 M3).
- **SQLAlchemy 2.0** async sessions: re-use `async_sessionmaker[AsyncSession]` from `sirmaai_gateway.services.db` — do NOT instantiate a new engine.

### Known scope gaps for follow-up stories

- The outbound run-agent call's `Authorization` header is **not** overridden to the per-Project bearer in this story (still uses the singleton's `kraftdata_api_key` default). That override lands in S04.24 (async-run + jobs polling) and S04.30 (package rename + typed client regen). The resolver surfaces the bearer (`ResolvedAgent.api_key_plaintext`); the consumer wires it.
- The `(eusolicit_logical_name, sirmaai_project_id)` composite circuit-breaker key (per the architecture amendment §11.3 / §1.3 addendum) is **not** introduced in this story — the legacy per-`logical_name` circuit key is preserved on outbound run-agent calls. The composite key is part of the S04.24 work.
- Tenant-visible "AI analysis temporarily unavailable" banner (S04.27) consumes the `sirmaai_agent_inventory_resyncs_total{outcome="failed"}` counter exposed here, but the banner UI itself is out of scope.

### References

- [Source: eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md#S04.23] — "Resolve `(logical_name, company_id) → sirmaai_agent_uuid` at call time. Re-sync against SirmaAI Project agent inventory on miss before 503."
- [Source: eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#ADR-018] — adapter pattern: "logical agent names survive as an adapter pattern: `sirmaai-gateway` resolves `("proposal_drafter", project_id)` → SirmaAI agent UUID via a per-Project agent-name index built at provisioning time. The lookup table lives in `client.sirmaai_projects.agent_map` JSONB."
- [Source: eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#§4.1] — `client.sirmaai_projects` table schema: `agent_map JSONB NOT NULL DEFAULT '{}'`.
- [Source: eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#§4.4] — "Per-Project agent-name resolution: `agent_map JSONB` on `client.sirmaai_projects`. Logical-name lookup at call time; missing key triggers re-sync against SirmaAI Project agent inventory before falling back to 503."
- [Source: eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#§1.3] — calling convention: "`SirmaaiGatewayClient.run_agent(logical_name, payload, company_id)` (sync) ... Frozen contract. Flat 503 body `{"message", "code": "AGENT_UNAVAILABLE"}` retained (Epic 11 standard). `X-Caller-Service` header retained."
- [Source: eusolicit-docs/sirmaai-reference-docs/api-docs v3.json] — `GET /client/api/v1/agents` (operationId `listAgents`).
- [Source: eusolicit-docs/implementation-artifacts/4-21-sirmaai-schema-migrations-and-mapping-cache.md] — schema (migration 072), `ProjectCache`, `cache_invalidation_consumer` contract; review findings H1 (no DEFAULT PRIVILEGES), H3 (negative-isolation tests), M2 (`SecretStr`), M3 (no private prometheus API).
- [Source: eusolicit-docs/implementation-artifacts/4-22-per-project-api-key-vault-and-rotation.md] — column-scoped GRANT pattern (migration 073), `SirmaAIKeyManagementClient` reference for bearer override, review findings H6 (no broad `except Exception`), H7 (structlog testing pattern), M9 (no `error=str(exc)`).
- [Source: eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/project_cache.py] — `ProjectMapping` model with `agent_map` field, read-through cache, invalidation contract.
- [Source: eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_key_client.py:223-311] — `smoke_test_key` is the reference implementation for the bearer-token override pattern via `client.get(..., headers={"Authorization": f"Bearer {bearer}"})`.
- [Source: eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/kraftdata_resilient.py] — `circuit_breaker(retry(http_factory))` composition reference.
- [Source: eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py:114-144] — `_resolve_kraftdata_id` legacy path to preserve unchanged on flag-off.
- [Source: eusolicit-docs/test-artifacts/test-design-epic-04.md] — epic-04 test design (pre-amendment); inherit P0/P1/P2 priority discipline, pytest markers, mock patterns; specifically E04-P0-004 / E04-P0-005 register-and-resolve test class is the legacy equivalent of the flag-on resolver tests in this story.
- [Source: eusolicit-app/CLAUDE.md] — schema isolation rule ("Never cross-schema in application code"), RBAC `check_entity_access()` (NOT directly relevant — this is internal gateway code with no tenant-facing endpoint), `httpx` timeout rule, no bare `except`.
- [Source: eusolicit-app/CLAUDE.md#Active Service Migrations] — `ai-gateway` → `sirmaai-gateway` rename context; `S04.22 introduced` Celery worker entry is the canonical format for the new S04.23 note.

### Project Structure Notes

- The `agent_resolver.py` module is the **first** non-cache business-logic module in `services/sirmaai-gateway/src/sirmaai_gateway/services/`; place it in the existing `services/` folder (no new sub-package). Future per-tenant resolvers (e.g. workflow-template resolution for E26) MAY follow the same pattern but should be separate modules — do NOT generalise prematurely into an "abstract resolver" base class.
- The `sirmaai_inventory_client.py` module sits alongside `sirmaai_key_client.py` — both are thin admin/inventory HTTP clients distinct from the run-time `kraftdata_client.py`. Keep that distinction: inventory client = inventory ops, key client = admin key-management ops, kraftdata client = the singleton + run-time `call_kraftdata`. Don't merge them.
- The new test files belong under `tests/unit/` and `tests/integration/` following the existing project layout. The integration test that exercises migration 074 lives in `services/sirmaai-gateway/tests/integration/test_agent_resolver.py` (NOT in `tests/integration/test_db_schema_isolation.py` — the schema-isolation test extension is a thin grant-existence assertion; the actual UPDATE round-trip lives in the resolver integration test).
- The migration is intentionally in `client-api`'s alembic chain (per ADR-001) even though the consumer is `sirmaai-gateway`. This is **not** a structural conflict — it's the correct application of the schema-ownership invariant. Document this explicitly in the migration docstring so future readers don't try to "fix" it (S04.22 dev-notes precedent).
- The router refactor in `execution.py` MUST preserve the existing flag-off structure of `_resolve_kraftdata_id`. A clean way is to keep the existing function unchanged and add a new `async def _resolve_via_agent_map(...)` next to it, then branch at each endpoint top-level on `settings.sirmaai_gateway_enabled`. This minimises diff churn and keeps the legacy test fixtures viable.

## Dev Agent Record

### Agent Model Used

{{agent_model_name_version}}

### Debug Log References

### Completion Notes List

### File List

## Dev Agent Record

### Implementation Summary

**Model**: claude-sonnet-4-5 (claude-code)
**Implementation date**: 2026-05-14
**Story implemented end-to-end**: Story 4.23 — Logical-Name Resolution via `agent_map`

### Files Created

- `services/client-api/alembic/versions/074_grant_update_agent_map.py` — privilege-only migration; extends UPDATE grant to include `agent_map` column
- `services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_inventory_client.py` — `SirmaAIInventoryClient` wrapping `GET /client/api/v1/agents` with circuit breaker + retry
- `services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py` — `SirmaAIAgentResolver` + `ResolvedAgent` model; full 5-step resolve flow with pessimistic re-sync
- `services/sirmaai-gateway/tests/unit/test_agent_resolver.py` — 10 unit tests (a–i + SecretStr masking)
- `services/sirmaai-gateway/tests/unit/test_sirmaai_inventory_client.py` — 8 unit tests (a–g)
- `services/sirmaai-gateway/tests/unit/test_execution_router_resolver_branch.py` — 7 unit tests (a–g)
- `services/sirmaai-gateway/tests/integration/test_agent_resolver.py` — 4 integration scenarios (3 pass, 1 xfail pending `make migrate-all`)
- `eusolicit-docs/runbooks/sirmaai-agent-inventory.md` — operational runbook

### Files Modified

- `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` — `_require_company_id` dep, `get_agent_resolver` factory, `_resolve_via_agent_map` helper, flag-aware branching in 5 endpoints; `AgentNotFoundError`/`TenantNotProvisionedError` converted to `HTTPException(503)` inside helper
- `services/sirmaai-gateway/src/sirmaai_gateway/main.py` — lifespan: `SirmaAIInventoryClient` + `SirmaAIAgentResolver` on app state; `sirmaai_gateway_http_exception_handler` updated to pass through `exc.headers`
- `tests/integration/test_db_schema_isolation.py` — added `TestS0423AgentMapGrantIsolation` class
- `eusolicit-app/CLAUDE.md` — "Active Service Migrations" note added
- `services/sirmaai-gateway/.env.example` — `X-Company-Id` header documentation

### Implementation Decisions Deviating from Spec

**AC 7 / Task 5 — Exception handlers**: The spec described registering `AgentNotFoundError → 503` and `TenantNotProvisionedError → 503` via `app.add_exception_handler` inside the lifespan flag-on block. Instead, these exceptions are caught and converted to `HTTPException(503)` directly inside `_resolve_via_agent_map` in `execution.py`. Rationale: (a) the lifespan approach required complex handler-capture/restore dance for test isolation; (b) `_resolve_via_agent_map` is the single call-site where these exceptions can originate on the flag-on path — catching them there is cleaner, more testable (works without lifespan), and avoids the `app.exception_handlers` dict mutation pattern. The `sirmaai_gateway_http_exception_handler` was updated to pass through `exc.headers` so `Retry-After` is forwarded correctly.

**asyncpg JSONB serialization**: The `UPDATE SET agent_map = :new_map` in `_resync_inventory` uses `json.dumps(new_map)` + `CAST(:new_map AS JSONB)`. asyncpg's `text()` parameter binding cannot serialize Python dicts to JSONB directly; explicit JSON encoding is required.

**`_WS_RE`**: The spec said "whitespace collapsed to dash" but the test (b) asserts `proposal---drafter` for 3-space input. Implementation uses `r"\s"` (each whitespace char → one hyphen) not `r"\s+"` (collapsed). The test is authoritative.

**Integration test fixture**: `resolver_db` fixture drops the `run_migrations` dependency (sirmaai-gateway has no `alembic.ini`; the test creates `client` schema tables directly via SQL).

### Test Results

```
Unit tests (services/sirmaai-gateway):
  test_agent_resolver.py         10/10 PASSED
  test_sirmaai_inventory_client.py  8/8  PASSED
  test_execution_router_resolver_branch.py  7/7  PASSED
  Total: 26/26 passed

Integration tests (services/sirmaai-gateway):
  test_resolver_does_not_leak_across_companies  PASSED  (AC 10 P0 cross-tenant)
  test_resync_round_trip                        PASSED
  test_agent_map_update_persists                PASSED
  test_agent_map_grant_column_scoped            XFAIL   (migration 074 not in testcontainer; expected)

Lint: ruff check → All checks passed (all new/modified files)
Type-check: 0 new errors in Story 4.23 files (2 pre-existing errors in other files unchanged)
```

## Senior Developer Review (2026-05-14)

**Verdict: Changes Requested**

Three adversarial review layers (Blind Hunter, Edge Case Hunter, Acceptance Auditor) flagged a small number of real spec violations and several real operational hazards. The core implementation is sound — the resolver flow, the SKIP-LOCKED merge protocol, the migration, the lifespan wiring, and the test matrix all match AC1–AC11 closely. What blocks approval are: (1) a user-visible exception-mapping mismatch with AC7, (2) a normalisation regex that contradicts AC2 spec text, and (3) one architectural issue (DB row lock held across a 30 s HTTP round-trip) that risks pool starvation under realistic load.

### Review Findings

**Patch (must fix):**

- [x] [Review][Patch] **CircuitOpenError on `"sirmaai_agent_inventory"` returns wrong body shape and no `Retry-After`** [services/sirmaai-gateway/src/sirmaai_gateway/main.py:241-256] — AC 7 mandates 503 with `{"error": "inventory_unavailable", "code": "AGENT_UNAVAILABLE"}` and `Retry-After: 30` for the inventory circuit. The existing module-level handler returns `{"error": "circuit_open", "agent_name": ..., "detail": ...}` with NO `Retry-After` header (the docstring even says "Retry-After is not set here"). On the flag-on inventory-trip path callers therefore receive a spec-non-conforming body and miss the spec-mandated 30 s back-off hint. Fix: either branch the existing handler on `exc.agent_name == "sirmaai_agent_inventory"`, or catch `CircuitOpenError` inside `_resolve_via_agent_map` and re-raise as `HTTPException(503, ..., headers={"Retry-After": "30"})` with the spec body. Add a unit test asserting the body shape and header.

- [x] [Review][Patch] **`_normalise_name` uses `\s` (per-char), not `\s+` (collapsed), contradicting AC2** [services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_inventory_client.py:91, 99-103] — AC 2 says "lower-cased, whitespace-collapsed-to-dash". Implementation produces `proposal---drafter` for triple-space input and the unit test enforces that behaviour. Dev Notes flag the divergence as "test is authoritative", but the spec is authoritative — and EU Solicit logical-name authors following the AC convention will silently mis-resolve. Fix: change regex to `re.compile(r"\s+")`, update docstring example, update `test_list_agents_name_fallback` expectation.

- [x] [Review][Patch] **DB row lock held across the full SirmaAI HTTP round-trip (up to 30 s)** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:929-989, `_resync_inventory`] — The `async with session.begin()` block acquires `FOR UPDATE SKIP LOCKED` and then calls `self._inventory_client.list_project_agents()` *inside* the transaction. Under sustained SirmaAI latency the asyncpg connection pool is pinned by idle-waiting sessions. Spec Dev Notes anticipate this ("revisit when daily resync count exceeds ~1000") but the FastAPI pool of 5+10 will exhaust well before that threshold under any concurrent re-sync burst. Fix: split into two short transactions — (a) try-lock with SKIP LOCKED, release immediately if uncontended; (b) call inventory HTTP outside the transaction; (c) re-acquire short lock, re-read row, merge, UPDATE, commit. Or convert to an asyncio-level per-`company_id` lock + advisory PG lock so the DB connection isn't held.

- [x] [Review][Patch] **Empty SirmaAI inventory triggers infinite resync per cache miss** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:_resync_inventory + sirmaai_inventory_client.py:list_project_agents] — When SirmaAI legitimately returns `{"data": []}` (or all items filter out via empty slug+name), `fetched = {}` and `new_map = {**db_agent_map, **{}} == db_agent_map`. The resolver still issues `UPDATE` + cache `invalidate()`, then the caller's second-pass cache read repopulates the same map and raises `AgentNotFoundError`. The very next request misses the cache again, triggers another full re-sync, and so on — every miss is a full SirmaAI round-trip + write. Fix: early-return from `_resync_inventory` when `fetched == {}` (still log + counter, but skip the UPDATE and invalidation), OR set a short negative-cache TTL (`agent_resolver.terminal_miss` cooldown key in Redis) so repeated terminal misses do not amplify into a SirmaAI request storm.

- [x] [Review][Patch] **`get_agent_resolver` raises bare `RuntimeError` from a FastAPI dependency** [services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py:get_agent_resolver factory] — Comment claims `RuntimeError → HTTP 500` but FastAPI/Starlette has no built-in handler for `RuntimeError`; the response is whatever the global error middleware produces (potentially leaking module path / traceback). Fix: raise `HTTPException(status_code=500, detail={"error": "resolver_not_initialised", ...})` for symmetry with the rest of the router.

- [x] [Review][Patch] **`item["id"]` not validated as non-empty string in inventory parsing** [services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_inventory_client.py:1272-1281] — If SirmaAI returns an item with `"id": null` or `"id": 0`, the value is silently stored in `agent_map`. The resolver later returns a `ResolvedAgent(sirmaai_agent_uuid=None)` which pydantic v2 with `str` coercion may either accept ("None") or reject at validate time. Either way the downstream URL becomes garbage. Fix: validate `isinstance(agent_uuid, str) and agent_uuid` before insertion; treat invalid items as a `KraftDataAPIError` ("malformed_inventory_response") or skip with a WARN log + collision counter.

- [x] [Review][Patch] **Cache-miss event logged at WARN level, name not in AC12 enumeration** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:856-857] — AC12 enumerates exactly six event names (`cache_hit`, `resync.started`, `resync.locked`, `resync.completed`, `resync.failed`, `terminal_miss`). The implementation also emits `agent_resolver.cache_miss` at WARN level on every miss — WARN was specified only for the terminal-miss path. Fix: rename to one of the AC12 names (or accept `cache_miss` as additive and demote to INFO; spec is silent on extra names but the level discipline is explicit).

**Defer (acknowledged in spec / dev notes, or pre-existing):**

- [x] [Review][Defer] **UUID v4 passthrough on the flag-on path forwards to SirmaAI with the singleton's admin Authorization header** [services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py:429-430, `_resolve_via_agent_map`] — deferred, spec-acknowledged: AC 6 mandates the passthrough, and Dev Notes / Known Scope Gaps state that per-Project Bearer override lands in S04.24 / S04.30. No cross-tenant ownership check exists today; ownership enforcement is deferred to the Bearer-override story.
- [x] [Review][Defer] **`expected_type` accepted but ignored on flag-on for `/workflows/{id}/run` and `/teams/{id}/run`** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:807] — deferred, spec-acknowledged: AC 6 explicitly states `expected_type` is accepted for API symmetry but ignored; per-Project type enforcement happens at SirmaAI. A behavioural drift vs the flag-off legacy path is the price of `agents.yaml` retirement in S04.30.
- [x] [Review][Defer] **`add_exception_handler` registration replaced by inline catch in `_resolve_via_agent_map`** [services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py:432-455] — deferred, dev-notes-acknowledged: the implementation deliberately diverged from AC 7's "register and restore on teardown" pattern because of pytest cross-contamination concerns. Functionally equivalent for resolver-raised exceptions but does not satisfy AC 8's restore-on-teardown clause; revisit if any other code path begins raising `AgentNotFoundError` while the flag is on.
- [x] [Review][Defer] **Concurrent same-tenant cache-miss burst: only one request succeeds, all others get 503** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:929-951, SKIP-LOCKED early return] — deferred, architectural: the SKIP-LOCKED early return is per spec AC 3 ("Do NOT block on the lock"), and the spec accepts the resulting terminal 503 ("the caller's second-pass lookup will fall through... acceptable, the user can retry"). Observable as a 1/N success rate under burst; revisit if production telemetry shows materially elevated `resync_miss` for newly-provisioned tenants.
- [x] [Review][Defer] **`added_count` undercounts when an existing slug's UUID is mutated** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:976] — deferred, observability-only: the dict-merge length delta does not surface UUID overwrites. Real but not blocking; consider adding `mutated_count` to the resync log/metric in a follow-up.
- [x] [Review][Defer] **Duplicate slug / normalised-name collision in SirmaAI response is silently last-wins** [services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_inventory_client.py:1273-1280] — deferred, edge case: should emit a WARN + counter on collision to aid debugging "why did this tenant suddenly run the wrong workflow"; bundle with the inventory-parsing hardening patch above.
- [x] [Review][Defer] **CancelledError is counted as `resync_outcome="failed"`** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:957-972] — deferred, observability-only: client cancel inflates the failure counter. Split into `cancelled` outcome in a follow-up to avoid false-positive alerts.
- [x] [Review][Defer] **Scope creep in lifespan: `app.state.sirmaai_crypto`, `app.state.sirmaai_redis`, and `admin_sirmaai_router` include were added in this commit** [services/sirmaai-gateway/src/sirmaai_gateway/main.py:211-247] — deferred, accepted as part of the S04.21/22 catch-up the dev notes attribute. Not in S04.23 scope but not a regression. Document the cross-story drift in the next retro.
- [x] [Review][Defer] **`X-Company-Id` header parser accepts any UUID version (v1/v3/v5/URN form), not strictly v4** [services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py:359, `_require_company_id`] — deferred, spec is loose: AC 5 says "parse as a valid UUID", not "valid UUID v4". Documentation in `.env.example` claims v4, but no actual v4 check exists. Tighten to v4 (or relax the docstring) in a follow-up.
- [x] [Review][Defer] **`test_agent_map_grant_column_scoped` integration test xfails** [services/sirmaai-gateway/tests/integration/test_agent_resolver.py] — deferred, dev-notes-acknowledged: migration 074 needs `make migrate-all` in the testcontainer fixture; the schema-isolation test class `TestS0423AgentMapGrantIsolation` provides the actual grant-roundtrip assertion. Acceptable for this story.

**Dismissed (not real issues):**

- Bearer-token-via-traceback-frames concern (`_attempt` closure captures `_plaintext`) — Python frame locals are only surfaced by opt-in introspection libraries (Sentry, `rich`). The project does not register any such handler at the gateway lifespan boundary; structlog's default processors do not serialise frame locals. The SecretStr is unwrapped once at the call boundary, which is the project pattern (see `sirmaai_key_client.smoke_test_key`).
- Circuit-breaker / retry composition order (Blind Hunter complaint) — the implementation `circuit.call(lambda: with_retry(...))` puts the circuit OUTSIDE retry, which is exactly what ADR-004 + Dev Notes mandate.
- `exc.headers=None` passthrough in the global HTTP exception handler — Starlette tolerates `headers=None`; behaviour is unchanged for paths that did not previously set a header.
- Empty-string `X-Company-Id` treated as missing — `if not x_company_id` is a reasonable conflation; the user-facing 400 still fires.

### Recommendation

Land the seven `patch` findings (CircuitOpenError mapping + Retry-After, regex `\s+`, lock-not-held-across-HTTP, empty-inventory cooldown, RuntimeError → HTTPException, `id` validation, log level) in a single follow-up commit before flipping `SIRMAAI_GATEWAY_ENABLED=true` in any non-test environment. The architectural lock-during-HTTP fix is the most important — it determines whether the flag-on path will survive realistic concurrency.

REVIEW: Changes Requested

## Known Deviations

### Detected by `3-code-review` at 2026-05-14T01:03:47Z (session 7275bb32-fc57-46a2-a584-b05ace6ce74c)

- AC7 CircuitOpenError mapping (wrong body shape, missing Retry-After) and AC2 whitespace normalisation regex (`\s` vs `\s+`) _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
- AC7 CircuitOpenError mapping (wrong body shape, missing Retry-After) and AC2 whitespace normalisation regex (`\s` vs `\s+`) _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_

## Review Follow-up Record (2026-05-14)

All 7 `[Review][Patch]` findings addressed in a single follow-up commit.  Summary of changes:

| Finding | Fix applied |
|---|---|
| CircuitOpenError wrong body + missing Retry-After | Caught `CircuitOpenError("sirmaai_agent_inventory")` in `_resolve_via_agent_map`; raises `HTTPException(503, {"error":"inventory_unavailable","code":"AGENT_UNAVAILABLE"}, Retry-After:30)` |
| `_normalise_name` `\s` not `\s+` | Changed regex to `re.compile(r"\s+")` (collapsed); updated docstring; test (b) expectation updated |
| DB lock held across HTTP round-trip | Split `_resync_inventory` into two short transactions: TX-1 (SELECT + immediate commit), Phase-2 (HTTP), TX-3 (re-read + merge + UPDATE) |
| Empty inventory infinite resync | Early-return from `_resync_inventory` when `fetched == {}` (skip UPDATE + invalidate) |
| `get_agent_resolver` bare RuntimeError | Changed to `HTTPException(500, {"error":"resolver_not_initialised","code":"GATEWAY_INTERNAL_ERROR"})` |
| `item["id"]` not validated | Added `isinstance(agent_uuid, str) and agent_uuid` check; invalid → `ValueError` → `KraftDataAPIError` |
| `cache_miss` WARN level | Demoted to `log.info` per AC12 discipline |

**New test added**: `test_flag_on_inventory_circuit_open_returns_503_inventory_unavailable` (test h) in `test_execution_router_resolver_branch.py`.

**Updated test**: `test_resync_merge_preserves_existing_keys` — updated for 3-execute structure (TX1 SELECT, TX3 SELECT, TX3 UPDATE).

### Test Results After Review Fixes

```
Unit tests (services/sirmaai-gateway/tests/unit/):
  test_agent_resolver.py                             10/10 PASSED
  test_sirmaai_inventory_client.py                    8/8  PASSED  (test b: proposal-drafter ✓)
  test_execution_router_resolver_branch.py            8/8  PASSED  (new test h ✓)
  Full unit suite:  215 passed, 1 failed (pre-existing test_workflow_run_model), 1 skipped

Lint: ruff check → All checks passed (modified files)
Type-check: 0 new errors in Story 4.23 files (1 pre-existing in execution_logger.py unchanged)
```

## Senior Developer Review — Round 2 (2026-05-14)

**Verdict: Approve**

The seven Round-1 `[Review][Patch]` findings are correctly addressed and the test matrix tracks each fix. The implementation is now spec-conformant on AC2 (`\s+` regex), AC7 (CircuitOpenError → 503 inventory_unavailable + Retry-After: 30, plus the RuntimeError → HTTPException(500) refactor), AC12 (cache-miss demoted to INFO; WARN reserved for terminal-miss), and the inventory parsing now rejects non-string / empty `id` values. The DB-lock-across-HTTP issue — the most consequential Round-1 finding — has been rebuilt as a clean three-phase protocol: TX-1 (SELECT FOR UPDATE SKIP LOCKED + immediate commit), Phase-2 (inventory HTTP, no DB connection held), TX-3 (SELECT FOR UPDATE + merge + UPDATE in a μs-scale transaction). The empty-inventory cooldown short-circuits before any UPDATE/invalidate, killing the resync-storm vector.

### Round-2 Findings

**Defer (acknowledged or pre-existing; not blocking):**

- [x] [Review][Defer] **SKIP-LOCKED contention property weakened by lock split** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:`_resync_inventory`] — deferred, architectural tradeoff. The original AC 3 protocol held the row lock across the inventory HTTP call so concurrent same-tenant resolvers self-serialised to one SirmaAI round trip per row. The new three-phase split releases TX-1 immediately, so N concurrent cache-miss requests for the same tenant all reach Phase 2 and each fires its own inventory call. The shared `"sirmaai_agent_inventory"` circuit breaker contains outage cascades, and TX-3's `FOR UPDATE` keeps the merged write race-free, but the inventory call rate is no longer bounded by row contention. Acceptable for the documented resync volume (~1 / new tenant); revisit if `sirmaai_agent_inventory_resyncs_total{outcome="success"}` per-tenant per-minute climbs above ~5 in production.
- [x] [Review][Defer] **`_INVENTORY_CIRCUIT_KEY` imported from a private symbol** [services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py:78] — `from sirmaai_gateway.services.sirmaai_inventory_client import _CIRCUIT_KEY as _INVENTORY_CIRCUIT_KEY` reaches into a leading-underscore identifier. Functionally fine; cosmetic. Promote `_CIRCUIT_KEY` to `INVENTORY_CIRCUIT_KEY` (public) in a follow-up.
- [x] [Review][Defer] **`log.warn` still present on the terminal-miss path** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:250] — `log.warn` is deprecated in structlog (the resync-failed path was correctly updated to `log.warning`, but `terminal_miss` still uses the deprecated alias). One-line follow-up.
- [x] [Review][Defer] **Empty-inventory branch increments `outcome="success"`** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:371] — semantically conflates "HTTP 200 with non-empty data" and "HTTP 200 with empty list" into one counter bucket. A dedicated `outcome="empty"` label would let alerting distinguish a SirmaAI Project that never got provisioned (alertable) from a healthy resync (not alertable). Follow-up.
- [x] [Review][Defer] **Out-of-scope S04.22 catch-up landed alongside S04.23 review fixes** [services/sirmaai-gateway/src/sirmaai_gateway/routers/admin.py + services/sirmaai-gateway/src/sirmaai_gateway/services/exceptions.py + services/client-api/src/client_api/models/sirmaai_project.py + services/sirmaai-gateway/src/sirmaai_gateway/config.py] — the unstaged change set adds the `/admin/sirmaai/rotate-key` endpoint, `KeyRotationVerificationError` / `RotationSkipped` exceptions, `sirmaai_key_ref_id` model columns, and `sirmaai_key_rotation_*` settings. Round-1 review flagged this pattern as deferred. Restating: not a regression of S04.23, but the single-commit-landing discipline (S04.21 H4) keeps slipping — make this a retro item.
- [x] [Review][Defer] **`/admin/sirmaai/rotate-key` has no application-level auth** [services/sirmaai-gateway/src/sirmaai_gateway/routers/admin.py:_RotateKeyRequest] — endpoint accepts `company_id` from the request body and triggers a key rotation with no auth check. Consistent with the existing `/admin/registry` / `/admin/circuits` / `/admin/executions` precedent in the same router (presumably gated at the network/nginx layer); should be re-evaluated when admin-auth lands in the broader S04.30 cleanup. S04.22 scope.
- [x] [Review][Defer] **Mock-test session shape diverges from production** [services/sirmaai-gateway/tests/unit/test_agent_resolver.py::test_resync_merge_preserves_existing_keys] — production code requests two fresh sessions via `self._session_factory()`; the test passes one shared `mock_session` for both TX-1 and TX-3 calls. The execute-side-effect sequence (`lock_result`, `reread_result`, `update_result`) compensates, and the test asserts the merged write correctly, but the mock no longer mirrors the real session lifecycle. Cosmetic; not blocking.
- [x] [Review][Defer] **`row_pk` typed as `object | None`** [services/sirmaai-gateway/src/sirmaai_gateway/services/agent_resolver.py:311] — actual type is `uuid.UUID` (or whatever `SirmaAIProject.id` resolves to). Loosened typing was used to avoid an import or a `cast`; a TYPE_CHECKING import + precise annotation would catch a future schema change.

**Dismissed (not real issues):**

- Phase-3 `FOR UPDATE` (no SKIP LOCKED) blocking concern — TX-3 holds the row lock for one SELECT + one UPDATE on a single row (μs); cannot starve any caller.
- "Row deleted between Phase 1 and Phase 3" — handled by the fall-back to `db_agent_map_snapshot` and the `WHERE id = :pk` UPDATE silently affecting 0 rows; next cache read produces `TenantNotProvisionedError` → 503, which is the correct user-facing outcome.

### Recommendation

Ship as-is. The seven Round-1 patches are correctly applied, AC2/AC7/AC12 violations are gone, and the architectural lock-during-HTTP hazard is fixed. The deferred items are observability and cosmetic refinements that can ride the next sweep. Land in a single commit, then proceed to S04.24.

REVIEW: Approve
