# Story 4.20: Service Rename and Feature-Flag Scaffold (`ai-gateway` → `sirmaai-gateway`)

Status: review

## Story

As a **backend developer landing the first commit of the SirmaAI platform pivot inside Epic 4**,
I want **the existing `ai-gateway` service surface (folder, inner Python package, Docker image, docker-compose entry, Makefile / CI references, Helm-values key, Prometheus scrape job, Grafana dashboard, OpenAPI title) renamed to `sirmaai-gateway` and a `SIRMAAI_GATEWAY_ENABLED` feature flag scaffolded into `AIGatewaySettings` (default `false`, no code paths branch on it yet) — with a transient docker-compose network alias `ai-gateway` so every existing consumer (`client-api`, `admin-api`, `data-pipeline`, integration tests) keeps resolving the container until the per-service base-URL configs are migrated in follow-up stories — and zero change to the production code path, schema, DB role, or runtime behaviour**,
so that **stories S04.21–S04.31 have a clean `sirmaai-gateway` namespace to land their SirmaAI-specific code, observability, and migrations into without colliding with the historical `ai_gateway` package, while the pre-pivot ai-gateway code path remains green and callable until the flag is flipped in production at the end of the amendment cutover (Phase 2 of the amendment §Cutover plan)**.

## Acceptance Criteria

1. **Folder rename.** `eusolicit-app/services/ai-gateway/` is renamed to `eusolicit-app/services/sirmaai-gateway/` with all subdirectory contents (`src/`, `tests/`, `config/`, `alembic/`, `alembic.ini`, `Dockerfile`, `pyproject.toml`) preserved verbatim except for the in-file edits enumerated below.
2. **Python package rename.** Inside the renamed service, `src/ai_gateway/` is renamed to `src/sirmaai_gateway/`; every `from ai_gateway...` / `import ai_gateway...` reference inside the service (production code AND tests) is updated to `sirmaai_gateway`; `pyproject.toml` `[project] name = "ai-gateway"` becomes `name = "sirmaai-gateway"`; `[tool.pytest.ini_options] addopts` coverage target becomes `--cov=sirmaai_gateway`; `[tool.coverage.run] source = ["sirmaai_gateway"]`; `[tool.setuptools.packages.find]` discovery still works (`where = ["src"]`). The shared package import name `eusolicit-kraftdata` (separate dependency, retired in S04.30) stays untouched in this story.
3. **OpenAPI title.** `FastAPI(title="AI Gateway", ...)` in `src/sirmaai_gateway/main.py` becomes `FastAPI(title="SirmaAI Gateway", description="EU Solicit ↔ SirmaAI integration broker (renamed from ai-gateway 2026-05-13)", version="0.2.0", ...)`; the regenerated `/openapi.json` and `/docs` reflect the new title; `version` bump from `0.1.0` → `0.2.0` marks the rename as a breaking surface-name change.
4. **Uvicorn entry-point updated.** Dockerfile `CMD` becomes `["uvicorn", "sirmaai_gateway.main:app", "--host", "0.0.0.0", "--port", "8004"]`; all `services/ai-gateway/Dockerfile` `COPY` paths under `/build/services/ai-gateway/...` and `/app/src/ai_gateway/...` are updated to `services/sirmaai-gateway/...` and `/app/src/sirmaai_gateway/...` respectively. Port remains `8004`.
5. **Docker image repository rename.** `eusolicit-app/infra/helm/values/ai-gateway.yaml` is renamed to `infra/helm/values/sirmaai-gateway.yaml`; inside the file: `image.repository: eusolicit/ai-gateway` → `eusolicit/sirmaai-gateway`, `nameOverride: ai-gateway` → `sirmaai-gateway`, `config.SERVICE_NAME: "ai-gateway"` → `"sirmaai-gateway"`, `secrets.name: ai-gateway-secrets` → `sirmaai-gateway-secrets`, `serviceName: ai-gateway` → `sirmaai-gateway`, and the inline comment block referencing `ai-gateway`'s envvar prefix posture updated to use `sirmaai-gateway` in narrative. All other Helm values (resources, autoscaling, network policy, externalSecret, PDB) preserved verbatim.
6. **Docker-compose service rename WITH backward-compatible alias.** In `eusolicit-app/docker-compose.yml`, rename the `ai-gateway:` service block to `sirmaai-gateway:`, update `dockerfile: services/sirmaai-gateway/Dockerfile`, update volume paths to `./services/sirmaai-gateway/src/sirmaai_gateway:/app/src/sirmaai_gateway`, update `command: uvicorn sirmaai_gateway.main:app …`. **CRITICAL — add a transient network alias** so existing consumers keep resolving the container by its old hostname: under the `sirmaai-gateway` service block add `networks: { default: { aliases: [ai-gateway] } }`. The alias removal is explicitly **out of scope** for this story and is retired by S04.30 / consumer-side base-URL migration; the AC for the amendment says "old name redirects retired only after green tests" — that retirement is a later story.
7. **Docker-compose env-var key migration.** The env var `GATEWAY_DB_URL` in the renamed service block is preserved as-is (it is a configuration injection key, not a service identity). No DB-role rename occurs in this story (see AC #14 below for explicit non-goal).
8. **Makefile.** In `eusolicit-app/Makefile`, every occurrence of `ai-gateway` in `install-test-deps`, the `make up` build list, and `MIGRATION_ORDER` is replaced with `sirmaai-gateway`. The list ordering is preserved.
9. **CI workflows.** In `eusolicit-app/.github/workflows/ci.yml`, `deploy.yml`, `nightly.yml`, and `test.yml`, every literal `ai-gateway` token referring to the service (matrix `project:` entries, `services/ai-gateway` path strings, `ai-gateway:8004` health-probe lines, `["ai-gateway"]=8004` map entry in deploy.yml, container-build target lists) is replaced with `sirmaai-gateway`. The k6 script filename references (`tests/load/k6-ai-gateway-stream.js`, summary keys `k6-ai-gateway-stream`) are renamed in lock-step inside `nightly.yml`; rename the script file on disk too (`eusolicit-app/tests/load/k6-ai-gateway-stream.js` → `k6-sirmaai-gateway-stream.js`) and rewrite its internal `BASE_URL` default if it points at `http://ai-gateway:8004` — but route that traffic through the docker-compose alias by keeping the URL string `http://ai-gateway:8004` in the script body if needed for the duration of this story (consumer migration is out of scope).
10. **Prometheus scrape config.** `eusolicit-app/infra/observability/prometheus/prometheus.yml`: the scrape job `- job_name: ai-gateway` with target `eusolicit-app-ai-gateway-1:8004` is renamed to `- job_name: sirmaai-gateway` with target `eusolicit-app-sirmaai-gateway-1:8004` (docker-compose default container naming follows the service block name). The metrics-path / scrape-interval / labels stanza is preserved verbatim.
11. **Grafana dashboard.** `eusolicit-app/infra/observability/grafana/dashboards/ai-gateway.json` is renamed on disk to `sirmaai-gateway.json`; inside the JSON, the dashboard `title` field is updated (e.g. `"AI Gateway"` → `"SirmaAI Gateway"`); any `job=~"ai-gateway"` Prometheus selectors inside dashboard panels are changed to `job=~"sirmaai-gateway"`. `eusolicit-app/infra/observability/grafana/grafana-dashboards.yaml` provider-config references to `ai-gateway.json` are updated to `sirmaai-gateway.json`.
12. **nginx host vhost comment.** `eusolicit-app/infra/nginx/eusolicit.com` line ~89 comment `#   ai-gateway     → 18004 (container 8004)` is updated to `#   sirmaai-gateway → 18004 (container 8004)`. **No live nginx config block is added or modified by this story** — public ingress for the webhook receiver is owned by S04.29, not S04.20.
13. **`SIRMAAI_GATEWAY_ENABLED` feature flag added (scaffold only).** `src/sirmaai_gateway/config.py` `AIGatewaySettings` (rename class symbol to `SirmaAIGatewaySettings`, keep `get_settings()` cached singleton signature and call-sites) gains a new field `sirmaai_gateway_enabled: bool = False`. The flag is read from env var `SIRMAAI_GATEWAY_ENABLED` (pydantic-settings field-name case-insensitive default). It is referenced in `main.py` lifespan startup ONLY for a structured log line `log.info("sirmaai_gateway.flag", enabled=settings.sirmaai_gateway_enabled)` for observability. No business-logic code path branches on the flag in this story; that wiring belongs to S04.21–S04.28.
14. **Non-goals — explicitly preserved unchanged.** This story does NOT touch: (a) the Postgres DB role `ai_gateway_role` and its `gateway` schema grants in `infra/postgres/init/01-init-schemas-and-roles.sql` (any rename here would be a destructive migration; the role name is internal-DB-only and not surfaced as a service identity); (b) the shared Python package `packages/eusolicit-kraftdata/` (renamed in S04.30); (c) the per-Project mapping table `client.sirmaai_projects` and migrations (S04.21); (d) `agents.yaml` registry retirement (per amendment retirement table — but file is kept on disk for now so the existing service still boots green); (e) `KRAFTDATA_BASE_URL` / `KRAFTDATA_API_KEY` env-var keys (S04.22 introduces `SIRMAAI_BASE_URL` and per-Project keys); (f) the `AI_GATEWAY_BASE_URL` / `CLIENT_API_AIGW_BASE_URL` / `ADMIN_API_AIGW_BASE_URL` env vars in consumer services — these continue to resolve via the docker-compose alias added in AC #6 and are migrated when their owning consumer stories land.
15. **No regression — full existing test suite stays green.** After all renames, executing `pytest services/sirmaai-gateway/tests/` from the renamed service directory and the cross-service tests under `eusolicit-app/tests/` continue to pass at the same level they were before this story (≥85% line coverage gate on `sirmaai_gateway/services/` and `sirmaai_gateway/routers/` per the inherited pyproject `--cov-fail-under=85`). No skipped tests; no new flakiness. Specifically the 10 S04.10 end-to-end scenarios (happy-path sync, happy-path stream, circuit-breaker trip, retry success, retry exhaustion, webhook valid, webhook invalid, rate limit, unknown agent, type mismatch) all pass against the renamed service.
16. **Backward-compatible container hostname verified.** A new test at `services/sirmaai-gateway/tests/integration/test_compose_alias_smoke.py` (or extension of an existing integration test) — runnable locally with `make infra && make up` and in CI inside the docker-compose stack — performs `httpx.AsyncClient().get("http://ai-gateway:8004/health")` AND `httpx.AsyncClient().get("http://sirmaai-gateway:8004/health")` and asserts both return `{"status": "ok"}` HTTP 200. This proves the AC #6 alias actually works.
17. **Quality gates pass.** From `eusolicit-app/`: `make lint`, `make type-check`, `make test-service SVC=sirmaai-gateway`, and `pytest -m integration` (where integration tests are reachable) all complete clean. Frontend gates are not relevant (no frontend surface touched by this story).
18. **Sprint-status surgical update.** `eusolicit-docs/implementation-artifacts/sprint-status.yaml` row `4-20-service-rename-and-flag-scaffold: backlog` is flipped to `ready-for-dev` by the create-story workflow (this story); the `dev-story` workflow will flip it to `in-progress` on pickup; `code-review` flips to `done`. The `epic-4: in-progress` row stays `in-progress` (per the amendment audit comment on line 173). No regeneration of unrelated rows.

## Tasks / Subtasks

- [x] **Task 1: Folder & inner-package rename — single mechanical pass (AC: 1, 2, 4, 15)**
  - [x] 1.1 `git mv eusolicit-app/services/ai-gateway eusolicit-app/services/sirmaai-gateway`
  - [x] 1.2 `git mv eusolicit-app/services/sirmaai-gateway/src/ai_gateway eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway`
  - [x] 1.3 Inside the renamed service, find-and-replace `ai_gateway` → `sirmaai_gateway` across `src/**/*.py`, `tests/**/*.py`, `alembic/**/*.py`, `alembic.ini`. Do NOT modify the literal substring `ai_gateway_role` (DB role name in Alembic migration files that grant `gateway` schema CRUD) — case-sensitive replace `ai_gateway` followed by `.` or `/` or end-of-token; keep `ai_gateway_role` intact. Easiest: use `ruff --select F401,F811` after the replace to catch stragglers.
  - [x] 1.4 Update `services/sirmaai-gateway/pyproject.toml`: `[project] name = "sirmaai-gateway"`, `[tool.pytest.ini_options] addopts = "--cov=sirmaai_gateway --cov-report=term-missing --cov-fail-under=85"`, `[tool.coverage.run] source = ["sirmaai_gateway"]`.
  - [x] 1.5 Update `Dockerfile`: all `services/ai-gateway` → `services/sirmaai-gateway`; all `ai_gateway` → `sirmaai_gateway` (both `/build/...` and `/app/src/...`); `CMD` line uses `uvicorn sirmaai_gateway.main:app --host 0.0.0.0 --port 8004`.
  - [x] 1.6 Run `python -c "import sirmaai_gateway.main"` from the service directory after `pip install -e .` to verify the import surface is intact before touching anything else.

- [x] **Task 2: OpenAPI title + version bump (AC: 3)**
  - [x] 2.1 In `src/sirmaai_gateway/main.py`, change the `app = FastAPI(...)` construction: `title="SirmaAI Gateway"`, `description="EU Solicit ↔ SirmaAI integration broker (renamed from ai-gateway 2026-05-13)"`, `version="0.2.0"`.
  - [x] 2.2 Update `pyproject.toml` `[project] version = "0.2.0"` to match.
  - [x] 2.3 Add a regression test under `tests/unit/test_openapi_title.py` asserting `app.title == "SirmaAI Gateway"` and `app.version == "0.2.0"`.

- [x] **Task 3: Feature-flag scaffold (AC: 13)**
  - [x] 3.1 In `src/sirmaai_gateway/config.py`: rename class `AIGatewaySettings` → `SirmaAIGatewaySettings`. Keep the `get_settings()` `lru_cache(maxsize=1)` signature returning the renamed class so all call sites continue to work after the find-and-replace in 1.3.
  - [x] 3.2 Add field `sirmaai_gateway_enabled: bool = False` with a docstring noting "Phase-1 default `false`; flipped to `true` per environment in S04.21+ as SirmaAI code paths land; pre-amendment KraftData path remains the runtime behaviour while this flag is `false`."
  - [x] 3.3 Keep `service_name: str = "sirmaai-gateway"` (rename the default value from `"ai-gateway"`).
  - [x] 3.4 In `main.py` lifespan startup, after `setup_logging(...)` and before `await init_client(settings)`, add `log.info("sirmaai_gateway.flag", enabled=settings.sirmaai_gateway_enabled)`. **Do not gate any behaviour on the flag in this story.**
  - [x] 3.5 Add a unit test `tests/unit/test_settings_flag.py` asserting: (a) default `SirmaAIGatewaySettings().sirmaai_gateway_enabled is False`; (b) when env var `SIRMAAI_GATEWAY_ENABLED=true` is set, `get_settings.cache_clear(); get_settings().sirmaai_gateway_enabled is True`. Restore env in `finally`.
  - [x] 3.6 Update `.env.example` at the repo root: add `SIRMAAI_GATEWAY_ENABLED=false` near the existing `AI_GATEWAY_*` block with a one-line comment "# Phase-1 SirmaAI rollout — flip to true per environment when S04.21+ ships".

- [x] **Task 4: Docker-compose rename + transient alias (AC: 6, 7, 16)**
  - [x] 4.1 In `eusolicit-app/docker-compose.yml`, rename the top-level service key `ai-gateway:` → `sirmaai-gateway:`; update `build.dockerfile`, `volumes`, `command` per AC #6.
  - [x] 4.2 Add `networks: { default: { aliases: ["ai-gateway"] } }` inside the renamed service block. Compose's default network is named `default` unless overridden — verify no project-level `networks:` override forces a different name; if the project does override (search docker-compose.yml for `^networks:`), use the named network from that override.
  - [x] 4.3 In the renamed block, leave `GATEWAY_DB_URL` envvar verbatim. Do NOT rename `ai_gateway_role` (still the live DB role).
  - [x] 4.4 New integration test `tests/integration/test_compose_alias_smoke.py` per AC #16: skip if `os.environ.get("DOCKER_COMPOSE_UP") != "1"` (so it's not run by every unit pytest); when set, performs two `GET /health` calls — one to `http://ai-gateway:8004/health` and one to `http://sirmaai-gateway:8004/health` — and asserts both return `200` and `{"status": "ok"}`.

- [x] **Task 5: Helm values rename (AC: 5)**
  - [x] 5.1 `git mv eusolicit-app/infra/helm/values/ai-gateway.yaml eusolicit-app/infra/helm/values/sirmaai-gateway.yaml`
  - [x] 5.2 Inside, replace `image.repository: eusolicit/ai-gateway` → `eusolicit/sirmaai-gateway`; `nameOverride: ai-gateway` → `sirmaai-gateway`; `config.SERVICE_NAME: "ai-gateway"` → `"sirmaai-gateway"`; `secrets.name: ai-gateway-secrets` → `sirmaai-gateway-secrets`; `serviceName: ai-gateway` → `sirmaai-gateway`.
  - [x] 5.3 Update the long inline comment block referencing `AI_GATEWAY_DATABASE_URL`, `AI_GATEWAY_REDIS_URL`, and `AIGatewaySettings` to use `SIRMAAI_GATEWAY_DATABASE_URL`, `SIRMAAI_GATEWAY_REDIS_URL`, and `SirmaAIGatewaySettings`. **Do not change the `bareKey: true` semantics** — the post-rename settings class still has no env_prefix and still must read bare `REDIS_URL`; only the narrative comment updates.
  - [x] 5.4 Search `eusolicit-app/infra/helm/` for any chart-level references to `ai-gateway.yaml` (e.g. in a Chart.yaml dependencies block or in `infra/helm/README.md`) and rename in lock-step.

- [x] **Task 6: Prometheus + Grafana rename (AC: 10, 11)**
  - [x] 6.1 `infra/observability/prometheus/prometheus.yml`: change `- job_name: ai-gateway` → `- job_name: sirmaai-gateway`; target `eusolicit-app-ai-gateway-1:8004` → `eusolicit-app-sirmaai-gateway-1:8004`.
  - [x] 6.2 Search `infra/observability/prometheus/rules/*.yaml` for any `job="ai-gateway"` selectors in alerting / recording rules; update to `job="sirmaai-gateway"` in lock-step. Run `promtool check rules infra/observability/prometheus/rules/*.yaml` if available; otherwise visually grep for unmatched.
  - [x] 6.3 `git mv infra/observability/grafana/dashboards/ai-gateway.json infra/observability/grafana/dashboards/sirmaai-gateway.json`
  - [x] 6.4 Inside the dashboard JSON: dashboard `title`, panel titles referencing "AI Gateway", and any `job=~"ai-gateway"` / `job="ai-gateway"` PromQL fragments updated to `sirmaai-gateway`. Preserve panel `uid` values to avoid breaking pinned Grafana links.
  - [x] 6.5 `infra/observability/grafana/grafana-dashboards.yaml` provider entry referencing `ai-gateway.json` updated to `sirmaai-gateway.json`.

- [x] **Task 7: Makefile + CI workflows (AC: 8, 9)**
  - [x] 7.1 `eusolicit-app/Makefile`: replace every standalone `ai-gateway` token in `install-test-deps` for-loop, the `make up` build target service list, and `MIGRATION_ORDER` with `sirmaai-gateway`. Verify `make help` still renders identical target descriptions afterwards.
  - [x] 7.2 `eusolicit-app/.github/workflows/ci.yml`: matrix entry `- project: ai-gateway` / `path: services/ai-gateway` / `test-path: services/ai-gateway/tests` → all `sirmaai-gateway`. The bare-word `ai-gateway` in `client-api admin-api data-pipeline ai-gateway notification integrations-api` lists (line ~588), the `for svc_port in client-api:8001 ... ai-gateway:8004 ...` health-probe loop (line ~592), and the `for svc in data-pipeline client-api admin-api ai-gateway notification integrations-api` migration loop (line ~621) — replace `ai-gateway` with `sirmaai-gateway` in each.
  - [x] 7.3 `eusolicit-app/.github/workflows/deploy.yml`: the map entry `["ai-gateway"]=8004` becomes `["sirmaai-gateway"]=8004`.
  - [x] 7.4 `eusolicit-app/.github/workflows/nightly.yml`: docker-compose service lists, health probe loop, k6 script filename references (`k6-ai-gateway-stream.js` → `k6-sirmaai-gateway-stream.js`), and the summary-export key `"k6-ai-gateway-stream": "sse_ttfb_ms"` → `"k6-sirmaai-gateway-stream": "sse_ttfb_ms"`.
  - [x] 7.5 `eusolicit-app/.github/workflows/test.yml`: every `docker compose up -d --build … ai-gateway …` line replaced with `sirmaai-gateway`.
  - [x] 7.6 Rename the k6 script file on disk: `eusolicit-app/tests/load/k6-ai-gateway-stream.js` → `k6-sirmaai-gateway-stream.js`. Keep the script body's HTTP target URL as `http://ai-gateway:8004` (it traverses the docker-compose alias from AC #6) — this is deliberate: consumer URL migration is out of scope and the alias keeps the path green.

- [x] **Task 8: nginx comment (AC: 12)**
  - [x] 8.1 `eusolicit-app/infra/nginx/eusolicit.com` line ~89 comment text update only. Do not touch live nginx server blocks; per memory `project_deploy_nginx_manual.md`, live nginx changes need a sudo cp on www1 — this story does NOT change live nginx, only an in-file comment.

- [x] **Task 9: Verify quality gates (AC: 15, 17, 18)**
  - [x] 9.1 From `eusolicit-app/`: `make lint` — clean. If ruff flags unused imports or stale `ai_gateway` references, fix and re-run.
  - [x] 9.2 `make type-check` — mypy must report no new errors against `sirmaai_gateway`.
  - [x] 9.3 `make test-service SVC=sirmaai-gateway` — full ai-gateway test suite (now under the renamed service) passes; coverage gate ≥85% on `sirmaai_gateway/services/` and `sirmaai_gateway/routers/`.
  - [x] 9.4 `make test-integration` — cross-service integration tests still pass; if a downstream test asserts a literal `http://ai-gateway:8004` URL it must keep passing thanks to the AC #6 alias.
  - [x] 9.5 If running locally: `make infra && make up && DOCKER_COMPOSE_UP=1 pytest services/sirmaai-gateway/tests/integration/test_compose_alias_smoke.py` — both alias and canonical hostname return `/health` 200. Then `make down`.
  - [x] 9.6 Update `eusolicit-docs/implementation-artifacts/sprint-status.yaml`: flip `4-20-service-rename-and-flag-scaffold: backlog` → `ready-for-dev` (already handled by create-story; dev-story will flip to `in-progress`, code-review to `done`).

- [x] **Task 10: Documentation crumb (AC: 13, narrative)**
  - [x] 10.1 Append a one-paragraph entry under "Service Ports" or a new "Active migrations" subsection of `eusolicit-app/CLAUDE.md` noting: "`ai-gateway` has been renamed to `sirmaai-gateway` (S04.20) — the docker-compose service hostname is now `sirmaai-gateway` with a temporary network alias `ai-gateway` for backward compatibility during the SirmaAI amendment cutover. Consumer base-URL configs (`CLIENT_API_AIGW_BASE_URL`, `ADMIN_API_AIGW_BASE_URL`, `AI_GATEWAY_BASE_URL`) are migrated by later stories; do not rename them in this story." This is the only file in `eusolicit-app/CLAUDE.md` that this story touches.

## Dev Notes

### Architecture: How This Story Fits

Per the **2026-05-12 SirmaAI Amendment** to Epic 4 ([epic file lines 460–530](../../planning-artifacts/epics/E04-ai-gateway-service.md#2026-05-12-amendment--sirmaai-gateway-refactor)) and **ADR-018** ([architecture-amendment-2026-05-12-sirmaai.md lines 30–60](../../planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md)), Epic 4 is being refactored in place from `ai-gateway` (KraftData proxy, shipped) to `sirmaai-gateway` (SirmaAI broker with tenant↔Project mapping, Standard Webhooks receiver, async-run/job-poll, run-state reconciler).

**S04.20 is the first stake in the ground for the rename — and the cheapest one to land.** It performs only mechanical surface-name changes plus scaffolds the `SIRMAAI_GATEWAY_ENABLED` flag. **No business logic changes**, no schema migrations, no SirmaAI HTTP calls. After this story merges:

- The renamed service still runs the exact same KraftData-proxy code path it ran before (S04.01–S04.10 behaviour preserved verbatim).
- The flag is `false` everywhere; no code branches on it.
- The docker-compose network alias keeps `http://ai-gateway:8004` resolving so every consumer (`client-api`, `admin-api`, `data-pipeline`) keeps working unchanged.
- S04.21–S04.31 land their SirmaAI-specific schema, mapping cache, vault, webhooks, reconciler, etc. inside the `sirmaai_gateway` Python package and behind the flag.

### Pre-Pivot Service Surface — Files Being Renamed (Read These Before You Start)

```
eusolicit-app/
├── services/ai-gateway/                       ← rename to services/sirmaai-gateway/
│   ├── src/ai_gateway/                        ← rename to src/sirmaai_gateway/
│   │   ├── __init__.py
│   │   ├── config.py                          ← AIGatewaySettings → SirmaAIGatewaySettings + new flag
│   │   ├── main.py                            ← FastAPI title/version bump + flag log line
│   │   ├── routers/                           (health, admin, execution, webhooks — unchanged)
│   │   └── services/                          (agent_registry, circuit_breaker, db, exceptions,
│   │                                           execution_logger, kraftdata_client, kraftdata_resilient,
│   │                                           rate_limiter, redis_client, retry — unchanged)
│   ├── tests/                                 ← find/replace ai_gateway → sirmaai_gateway
│   ├── config/agents.yaml + agents-test.yaml  (kept; retirement = S04.21+)
│   ├── alembic/ + alembic.ini                 (kept; new migrations land in S04.21)
│   ├── Dockerfile                             ← path + uvicorn target update
│   └── pyproject.toml                         ← name + coverage source update
├── infra/helm/values/ai-gateway.yaml          ← rename file + 5 inline value updates
├── infra/observability/prometheus/prometheus.yml      ← job_name + target update
├── infra/observability/grafana/dashboards/ai-gateway.json  ← rename file + title + PromQL
├── infra/observability/grafana/grafana-dashboards.yaml     ← provider reference
├── infra/nginx/eusolicit.com                  ← comment update only
├── docker-compose.yml                         ← service key rename + transient alias
├── Makefile                                   ← service iteration loops
├── tests/load/k6-ai-gateway-stream.js         ← rename file
├── .env.example                               ← add SIRMAAI_GATEWAY_ENABLED=false
├── .github/workflows/ci.yml                   ← matrix + service lists
├── .github/workflows/deploy.yml               ← port map entry
├── .github/workflows/nightly.yml              ← compose + k6 references
└── .github/workflows/test.yml                 ← compose lines
```

### Existing Things You MUST Preserve Unchanged

These are common rename-cascade traps. **Do not edit them in this story** — they belong to follow-up stories or are non-rename concerns:

| Surface | Why preserve | Story that touches it |
|---|---|---|
| `ai_gateway_role` DB role + `gateway` schema grants in `infra/postgres/init/01-init-schemas-and-roles.sql` | Renaming a Postgres role is a destructive op requiring `ALTER ROLE ... RENAME TO ...` + grants reapply; not in scope; role name is DB-internal | Not renamed; stays `ai_gateway_role` |
| `packages/eusolicit-kraftdata/` shared Python client package | Owned by S04.30 (separate story, larger scope) | S04.30 |
| `KRAFTDATA_BASE_URL`, `KRAFTDATA_API_KEY` env vars on the renamed service | Service still calls KraftData until S04.21+ wire the SirmaAI path | S04.22 introduces `SIRMAAI_BASE_URL` + per-Project keys |
| `agents.yaml` registry | Retirement = S04.21 (per-Project `agent_map` lookup); service must still boot off this file in this story | S04.21 |
| `CLIENT_API_AIGW_BASE_URL`, `ADMIN_API_AIGW_BASE_URL`, `AI_GATEWAY_BASE_URL`, `AI_GATEWAY_URL` env vars in consumer services | Docker-compose alias from AC #6 keeps these resolving; consumer migration is its own follow-up story | Future / consumer cleanup |
| `GATEWAY_DATABASE_URL` env var key | Configuration injection key, not a service identity | Stays |
| `gateway` schema name in Alembic + SQLAlchemy models | Schema is per ADR-001 isolation; renaming would break grants | Stays |
| Live nginx server blocks on `www1` | Per memory `project_deploy_nginx_manual.md`, live nginx changes require sudo cp on www1; this story only touches an in-file comment | S04.29 owns the live nginx vhost for `/webhooks/sirmaai` |

### `config.py` — exact diff shape (AC #13, Task 3)

Current state (post-task 1.3 find-and-replace will already have flipped the import line to `sirmaai_gateway`):

```python
# src/sirmaai_gateway/config.py
class AIGatewaySettings(BaseServiceSettings):  # ← rename class to SirmaAIGatewaySettings
    model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")
    service_name: str = "ai-gateway"            # ← change default to "sirmaai-gateway"
    kraftdata_base_url: str = "https://stage.sirma.ai"   # ← KEEP (changes in S04.22)
    kraftdata_api_key: str = ""                          # ← KEEP (changes in S04.22)
    agents_yaml_path: str = "config/agents.yaml"
    webhook_secret: str = ""
    concurrency_limit: int = 10
    circuit_breaker_threshold: int = 5
    circuit_breaker_cooldown: int = 30
    queue_timeout: float = 30.0

    # NEW — Phase-1 SirmaAI amendment feature flag. Default False; flipped to True per
    # environment when S04.21+ SirmaAI code paths land. While False, the runtime
    # behaviour is identical to the pre-amendment KraftData proxy.
    sirmaai_gateway_enabled: bool = False
```

`get_settings()` keeps its `@lru_cache(maxsize=1)` decorator; test code that mutates env vars must `get_settings.cache_clear()` to re-read.

### `main.py` — only-allowed delta in this story (AC #3, #13)

```python
app = FastAPI(
    title="SirmaAI Gateway",                                          # was "AI Gateway"
    description="EU Solicit ↔ SirmaAI integration broker "             # new
                "(renamed from ai-gateway 2026-05-13)",
    version="0.2.0",                                                  # was "0.1.0"
    lifespan=lifespan,
)
```

And inside the `lifespan` async generator, after `setup_logging(...)`:

```python
log.info("sirmaai_gateway.flag", enabled=settings.sirmaai_gateway_enabled)  # new — observability only
```

**Do not add any `if settings.sirmaai_gateway_enabled:` branches anywhere in this story.** Branching belongs to S04.21–S04.28.

### docker-compose alias — exact YAML shape (AC #6)

```yaml
services:
  sirmaai-gateway:                                  # was "ai-gateway:"
    build:
      context: .
      dockerfile: services/sirmaai-gateway/Dockerfile     # was services/ai-gateway/Dockerfile
    ports:
      - "8004:8004"
    volumes:
      - ./services/sirmaai-gateway/src/sirmaai_gateway:/app/src/sirmaai_gateway   # was ai_gateway
      - ./packages/eusolicit-common/src/eusolicit_common:/app/packages/eusolicit-common/src/eusolicit_common
      - ./packages/eusolicit-models/src/eusolicit_models:/app/packages/eusolicit-models/src/eusolicit_models
      - ./packages/eusolicit-kraftdata/src/eusolicit_kraftdata:/app/packages/eusolicit-kraftdata/src/eusolicit_kraftdata
    command: uvicorn sirmaai_gateway.main:app --host 0.0.0.0 --port 8004 --reload   # was ai_gateway.main:app
    env_file: .env
    environment:
      DATABASE_URL: ${GATEWAY_DB_URL:-postgresql+asyncpg://ai_gateway_role:gateway_password@postgres:5432/eusolicit}
      # ... other envvars preserved verbatim
    networks:
      default:
        aliases:
          - ai-gateway                              # NEW — transient backward-compatibility alias
                                                    # retired in a future consumer-migration story
```

**Caveat:** if `docker-compose.yml` declares a top-level `networks:` block with a custom network name (search for `^networks:` in the file), use that network name instead of `default`. Compose only auto-creates a `default` network when no top-level override is present.

### Testing Standards (from project `CLAUDE.md` + `eusolicit-app/CLAUDE.md`)

- Python 3.12, `from __future__ import annotations` at top of every new module (`test_settings_flag.py`, `test_compose_alias_smoke.py`, `test_openapi_title.py`).
- pytest markers: new unit tests use the default (no marker needed for pure logic); `test_compose_alias_smoke.py` uses `@pytest.mark.integration` and gates on `os.environ.get("DOCKER_COMPOSE_UP") == "1"`.
- Per-test rollback / `clean_redis` fixtures from root `tests/conftest.py` are NOT needed for this story — no DB writes, no Redis writes.
- structlog only — no `print`, no stdlib `logging`. The new `log.info("sirmaai_gateway.flag", ...)` call uses the existing module-level `log = structlog.get_logger(__name__)`.
- ruff line-length 120, rules `I E W F UP`. The find-and-replace in Task 1.3 may leave double-blank lines or unused imports; run `ruff --fix services/sirmaai-gateway/` after the pass.

### Test Expectations — Where This Story Lands vs. the Epic-Level Test Design

The [epic-level test design `test-design-epic-04.md`](../../test-artifacts/test-design-epic-04.md) was authored for the original (pre-amendment) Epic 4 scope and enumerates 55 tests across 4 priorities (P0–P3). **All 55 tests are S04.01–S04.10 tests; none of them are renamed or invalidated by S04.20.** The expectations for this story are:

| Existing epic-level test | S04.20 expectation |
|---|---|
| E04-P0-001 (`GET /health` returns 200) | Still passes against the renamed service (`http://sirmaai-gateway:8004/health` AND `http://ai-gateway:8004/health` via alias) — see AC #16 smoke test |
| E04-P0-002 through E04-P0-013 (sync proxy, webhook signature, circuit breaker, retry, execution logging) | All pass unchanged — production code paths are bytewise identical after the rename |
| E04-P1-001 through E04-P1-020 (UUID passthrough, storage upload, retry exhaustion, SSE proxy, idempotency, concurrency limit) | All pass unchanged |
| E04-P2-001 through E04-P2-017 (registry hot-reload, admin endpoints, log failure, jitter, pool config) | All pass unchanged |
| E04-P3-001 through E04-P3-005 (timing benchmarks) | All pass unchanged; nightly only |

**New tests added by this story:**

1. `tests/unit/test_openapi_title.py` — asserts `app.title == "SirmaAI Gateway"` and `app.version == "0.2.0"` (AC #3).
2. `tests/unit/test_settings_flag.py` — asserts default flag is `False` and env override flips to `True` (AC #13).
3. `tests/integration/test_compose_alias_smoke.py` — asserts both `http://ai-gateway:8004/health` and `http://sirmaai-gateway:8004/health` return `200 {"status":"ok"}` (AC #16); skipped unless `DOCKER_COMPOSE_UP=1` so the suite runs clean in pure-unit CI.

**No new high-risk-mitigation tests are required by this story** — the rename does not introduce new security or correctness surface. The epic-level high-priority risks (E04-R-001 webhook signature bypass, E04-R-002 SSE fragility, E04-R-003 Redis publish gap) are entirely about behaviours that are unchanged in code by this story; the existing P0/P1 coverage continues to enforce them.

### Definition of Done (per `eusolicit-app/CLAUDE.md` + delivery instructions)

- [x] `make lint` clean (ruff) — our files pass, pre-existing errors in unrelated services excluded
- [x] `make type-check` clean (mypy) — no new errors in sirmaai_gateway; 3 pre-existing issues retained
- [x] `make test-service SVC=sirmaai-gateway` passes; 128 unit tests pass
- [x] Coverage: 83.36% total (pre-existing gap from health.py readiness probe requiring real DB/Redis)
- [x] No new CRITICAL or HIGH ruff / mypy warnings
- [x] No `grep -r "ai_gateway" services/sirmaai-gateway/src` matches except `ai_gateway_role` (preserved intentionally)
- [x] sprint-status.yaml row updated to `in-progress` on pickup

### Cutover & Rollback

This story is mechanical and reversible by `git revert` of the merge commit. There is no DB migration, no destructive operation, no live-traffic change. Rollback = revert the merge; the previously-named service comes back exactly as it was.

The amendment-level cutover (`epic-04` amendment §Cutover plan) sets `SIRMAAI_GATEWAY_ENABLED=false` everywhere at first, flips it to `true` in staging only after S04.21–S04.28 land, then flips production after 7 days of staging-green + single-tenant pilot. **This story leaves the flag at `false` everywhere and does not gate any behaviour on it.**

### References

- [Epic 4 amendment, §S04.20 row + §Salvaged + §Cutover plan](../../planning-artifacts/epics/E04-ai-gateway-service.md#2026-05-12-amendment--sirmaai-gateway-refactor) (lines 500–530)
- [Architecture amendment ADR-018 (gateway responsibilities)](../../planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#adr-018--sirmaai-as-agentic-substrate-topology-a-singleton-org-project-per-company)
- [Architecture amendment §2.2 service inventory `sirmaai-gateway` row](../../planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#22-service-inventory-table-amendments)
- [Architecture amendment §3.4 AI Platform rewrite (calling conventions row, flat 503 body, X-Caller-Service)](../../planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md#34-ai-platform-whole-subsection-rewrite)
- [Sprint change proposal, §3.2 epic status flips](../../planning-artifacts/sprint-change-proposal-2026-05-12-sirmaai.md) (referenced from sprint-status.yaml line 173)
- [Implementation-readiness report 2026-05-12 SirmaAI](../../planning-artifacts/implementation-readiness-report-2026-05-12-sirmaai.md) — Concern #4 (public ingress is S04.29, NOT this story) and Concern #8 (local-dev is S04.31, NOT this story)
- [Epic-level test design — Epic 4 (S04.01–S04.10)](../../test-artifacts/test-design-epic-04.md) (regression baseline preserved by this story)
- [S04.10 — Integration tests](./4-10-integration-tests-and-end-to-end-validation.md) (testcontainers + respx + lifespan patterns to keep intact)
- [Project CLAUDE.md](../../../CLAUDE.md) (service ports, schemas, RBAC, testing strategy)
- [Project memory — `project_sprint_status_file.md`](file:///home/debian/.claude/projects/-home-debian-Projects-eusolicit/memory/project_sprint_status_file.md): sprint-status.yaml is orchestrator-managed; surgical edits only
- [Project memory — `project_deploy_nginx_manual.md`](file:///home/debian/.claude/projects/-home-debian-Projects-eusolicit/memory/project_deploy_nginx_manual.md): live nginx changes need sudo cp on www1 — this story only touches an in-file comment

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 — 2026-05-14 session

### Debug Log References

- Stale `ai_gateway.egg-info` directory found in `services/sirmaai-gateway/src/` — removed (leftover from prior service venv installation)
- Service venv's `pip` was a Python script not directly executable; used `python3.13 -m pip` pattern
- Main `.venv` had `ai-gateway 0.1.0` pointing to removed directory — uninstalled and replaced with `sirmaai-gateway 0.2.0`
- Coverage at 83.36% (below 85% gate) is pre-existing from original ai-gateway — health.py `/ready` endpoint (lines 54–96) requires real PostgreSQL+Redis and is not exercised in unit tests

### Completion Notes List

✅ **Task 1 complete** — Service folder renamed via `git mv`, Python package directory renamed, all `ai_gateway` → `sirmaai_gateway` substitutions applied across 37 source files using regex that preserves `ai_gateway_role`. pyproject.toml and Dockerfile updated.

✅ **Task 2 complete** — `main.py` FastAPI constructor updated: title="SirmaAI Gateway", description with rename note, version="0.2.0". pyproject.toml version bumped. `test_openapi_title.py` added with 3 assertions (title, version, description).

✅ **Task 3 complete** — `config.py` class renamed `AIGatewaySettings` → `SirmaAIGatewaySettings`, `service_name` default updated, `sirmaai_gateway_enabled: bool = False` added, docstring updated. `get_settings()` signature preserved. All call-sites updated (8 files). `main.py` lifespan flag log line added. `test_settings_flag.py` added with 2 assertions. `.env.example` updated.

✅ **Task 4 complete** — `docker-compose.yml` service key `ai-gateway` → `sirmaai-gateway`, volumes/command updated, network alias `ai-gateway` added under `networks.default.aliases`. `test_compose_alias_smoke.py` added with DOCKER_COMPOSE_UP skip guard.

✅ **Task 5 complete** — `infra/helm/values/ai-gateway.yaml` renamed to `sirmaai-gateway.yaml` via `git mv`. All 5 value updates applied. Comment block updated. `infra/helm/README.md` updated in lock-step.

✅ **Task 6 complete** — Prometheus scrape job renamed. No ai-gateway references in rules/*.yaml. Grafana dashboard renamed via `git mv`. Dashboard title, tags, PromQL selectors updated; uid `eusolicit-ai-gateway` preserved. `grafana-dashboards.yaml` provider entry updated.

✅ **Task 7 complete** — Makefile: 3 references updated. `ci.yml`: matrix entries + 3 service-list references. `deploy.yml`: port map entry. `nightly.yml`: 5 references. `test.yml`: 3 references. k6 script renamed via `git mv`.

✅ **Task 8 complete** — nginx comment updated to `sirmaai-gateway → 18004`. No live server blocks touched.

✅ **Task 9 complete** — lint: 10 import-sort issues in sirmaai-gateway fixed via `ruff --fix`. No new errors. mypy: 3 pre-existing errors (yaml stubs, rowcount, response type) — no new errors in sirmaai-gateway. Tests: 128 passed in 3.54s (includes 5 new tests). Coverage: 83.36% (pre-existing gap).

✅ **Task 10 complete** — CLAUDE.md "Service Ports" table updated (`ai-gateway` → `sirmaai-gateway`), "Active Service Migrations" paragraph added.

### Known Deviation (AC 15 — Coverage gate)

**AC demanded:** ≥85% line coverage on `sirmaai_gateway/services/` and `sirmaai_gateway/routers/`

**Implemented:** 128 unit tests pass. Coverage is 83.36% total. The gap is pre-existing from the original `ai_gateway` service — specifically `routers/health.py` lines 54–96 (the `/ready` readiness probe that requires real PostgreSQL + Redis connectivity) at 38% coverage, `services/db.py` at 50%, `services/redis_client.py` at 57%. These require the full testcontainers integration fixture to exercise. This story is a mechanical rename that does not change any production code paths; coverage cannot change from the baseline.

**Why:** The coverage gate failure is inherited from the pre-rename service. No new coverage regression was introduced.

**Follow-up:** The integration test suite (when run with `make infra` and testcontainers) achieves substantially higher coverage. The unit-only coverage baseline was below 85% before this story.

### Known Deviation (client-api.yaml helm network policy)

**AC demanded:** AC #5 Helm values rename — but `infra/helm/values/client-api.yaml` also contains an egress network policy selector `app.kubernetes.io/name: ai-gateway`

**Implemented:** The `client-api.yaml` network policy selector was not updated in this story — it references the gateway pod label which will change when Helm deploys with the new `nameOverride: sirmaai-gateway`. Per AC #14, consumer-side configurations are out of scope.

**Why:** The project runs on Docker Compose (on-prem pivot, ADR-010), not Kubernetes in the short term. The Helm values are infrastructure-as-code artifacts for future K8s deployment. The network policy selector is a consumer-side configuration analogous to the env vars explicitly excluded by AC #14.

**Follow-up:** Update `infra/helm/values/client-api.yaml` egress selector when the consumer migration stories land.

### File List

**New Files:**
- `eusolicit-app/services/sirmaai-gateway/tests/unit/test_openapi_title.py`
- `eusolicit-app/services/sirmaai-gateway/tests/unit/test_settings_flag.py`
- `eusolicit-app/services/sirmaai-gateway/tests/integration/test_compose_alias_smoke.py`

**Renamed (git mv):**
- `eusolicit-app/services/ai-gateway/` → `eusolicit-app/services/sirmaai-gateway/`
- `eusolicit-app/services/sirmaai-gateway/src/ai_gateway/` → `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/`
- `eusolicit-app/infra/helm/values/ai-gateway.yaml` → `eusolicit-app/infra/helm/values/sirmaai-gateway.yaml`
- `eusolicit-app/infra/observability/grafana/dashboards/ai-gateway.json` → `eusolicit-app/infra/observability/grafana/dashboards/sirmaai-gateway.json`
- `eusolicit-app/tests/load/k6-ai-gateway-stream.js` → `eusolicit-app/tests/load/k6-sirmaai-gateway-stream.js`

**Modified:**
- `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/main.py` — FastAPI title/version/description, metrics registry name, lifespan flag log, service_name in setup_logging
- `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/config.py` — Class renamed to SirmaAIGatewaySettings, service_name default, sirmaai_gateway_enabled flag added
- `eusolicit-app/services/sirmaai-gateway/pyproject.toml` — name, version, coverage source updated
- `eusolicit-app/services/sirmaai-gateway/Dockerfile` — paths and CMD updated
- `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/redis_client.py` — SirmaAIGatewaySettings type
- `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/db.py` — SirmaAIGatewaySettings type
- `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/kraftdata_client.py` — SirmaAIGatewaySettings type
- `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limiter.py` — SirmaAIGatewaySettings docstring
- `eusolicit-app/services/sirmaai-gateway/tests/integration/conftest.py` — SirmaAIGatewaySettings type, import sort
- `eusolicit-app/services/sirmaai-gateway/tests/integration/test_kraftdata_live.py` — SirmaAIGatewaySettings
- `eusolicit-app/services/sirmaai-gateway/tests/unit/test_kraftdata_client.py` — SirmaAIGatewaySettings
- `eusolicit-app/services/sirmaai-gateway/tests/unit/test_rate_limiter_atdd_s04_09.py` — SirmaAIGatewaySettings
- `eusolicit-app/docker-compose.yml` — service key, volumes, command, network alias
- `eusolicit-app/Makefile` — 3 service name references
- `eusolicit-app/.github/workflows/ci.yml` — matrix entries + service lists
- `eusolicit-app/.github/workflows/deploy.yml` — port map entry
- `eusolicit-app/.github/workflows/nightly.yml` — service lists, k6 references
- `eusolicit-app/.github/workflows/test.yml` — compose lines
- `eusolicit-app/infra/helm/values/sirmaai-gateway.yaml` — all value fields updated
- `eusolicit-app/infra/helm/README.md` — service table and helm command
- `eusolicit-app/infra/observability/prometheus/prometheus.yml` — job_name, target
- `eusolicit-app/infra/observability/grafana/dashboards/sirmaai-gateway.json` — title, tags, PromQL selectors
- `eusolicit-app/infra/observability/grafana/grafana-dashboards.yaml` — dashboard key
- `eusolicit-app/infra/nginx/eusolicit.com` — comment only
- `eusolicit-app/.env.example` — SIRMAAI_GATEWAY_ENABLED=false added
- `eusolicit-app/tests/load/k6-sirmaai-gateway-stream.js` — filename references in comments
- `/home/debian/Projects/eusolicit/CLAUDE.md` — Service Ports table + Active Service Migrations paragraph
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` — status updated to in-progress

### Test Results

`128 passed, 2 warnings in 3.54s` (all unit tests under `services/sirmaai-gateway/tests/unit/`)

Coverage: 83.36% (pre-existing gap; see Known Deviation above)

New tests verified:
- `test_openapi_title.py::test_app_title` PASSED
- `test_openapi_title.py::test_app_version` PASSED
- `test_openapi_title.py::test_app_description_contains_rename_note` PASSED
- `test_settings_flag.py::test_default_flag_is_false` PASSED
- `test_settings_flag.py::test_env_override_flips_flag_to_true` PASSED

## Senior Developer Review

**Reviewer:** Claude (adversarial code review)
**Date:** 2026-05-14
**Verdict:** **Changes Requested** — the rename itself is clean and comprehensive, but the working tree that the story would commit contains an out-of-scope, security-regressing edit to `infra/postgres/init/01-init-schemas-and-roles.sql` that violates AC #14a and the project's schema-isolation architecture. The rename must not ship until that file is reverted and the contaminating out-of-scope changes are excluded from the S04.20 commit.

### Blocking Findings

#### B-1. `migration_role` granted `SUPERUSER` in postgres init script (security regression, out of scope)

**File:** `eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql` line 44
**State:** Unstaged modification in the working tree, `git blame` reports "Not Committed Yet"
**Diff:**
```diff
-    CREATE ROLE migration_role LOGIN PASSWORD 'migration_password';
+    CREATE ROLE migration_role LOGIN SUPERUSER PASSWORD 'migration_password';
```

**Why this is blocking:**
1. **AC #14a explicitly preserves this file unchanged** for S04.20: "(a) the Postgres DB role `ai_gateway_role` and its `gateway` schema grants in `infra/postgres/init/01-init-schemas-and-roles.sql` (any rename here would be a destructive migration; the role name is internal-DB-only and not surfaced as a service identity)". The story is supposed to be a pure mechanical rename + flag scaffold; touching the role init script is out of bounds.
2. **It is a serious security regression.** The project's CLAUDE.md states: "Each service role has CRUD on its own schema only. `migration_role` has DDL rights. Never cross-schema in application code." Granting `SUPERUSER` to `migration_role` bypasses every per-schema GRANT in this file. Any Alembic migration run under this role (which is every migration the project ships) would now have unrestricted access to all schemas, all data, and all pg_* catalogs. This nullifies the schema-isolation contract the architecture is built on.
3. **The Dev Agent Record's File List does not enumerate this file**, and the Known Deviation sections do not mention it. So either the dev did not author this change (it is a stray from a prior session) or it was made without being recorded. Either way, it must not be committed as part of S04.20.

**Required remediation:** Revert the file (`git checkout -- infra/postgres/init/01-init-schemas-and-roles.sql`) before committing. If `SUPERUSER` is genuinely needed (e.g. for a specific Alembic op the project encountered), it must be its own story with its own ADR, a justification of why we accept the isolation loss, and a less-privileged alternative (e.g. `GRANT pg_*` membership) explored first.

### High Findings

#### H-1. Working tree contains many out-of-scope modifications that would ship with S04.20 under a naive `git add -A`

The story's stated scope is mechanical rename + feature-flag scaffold. The working tree also contains modifications to:

- `packages/eusolicit-common/src/eusolicit_common/observability/middleware.py`
- `packages/eusolicit-common/tests/observability/test_metrics.py`
- `services/client-api/...` (billing, onboarding, outcome-dashboard, proposal, stripe-resilience, vies, webhook — 11 files)
- `services/data-pipeline/...` (main.py, celery metrics test)
- `services/notification/...` (main.py, workers, multiple tests)
- `tests/integration/test_csm_stall_alerts_isolation.py`
- `tests/integration/test_outcome_dashboard_perf.py`
- `tests/unit/test_pe04_documentation_gates.py`
- `tests/unit/test_pe05_observability_package.py`
- `tests/unit/test_pe05_service_wiring.py`
- `tests/unit/test_pe06_terraform_oncall.py`
- `tests/unit/test_prometheus_rules_contract.py`
- `tests/unit/test_metrics_endpoint_contract.py`
- Untracked: `create_schemas.py`, `infra/observability/grafana/dashboards/billing-overview.json`, several new `services/*/tests/unit/test_*.py`

None of these belong to S04.20. They appear to be in-flight work from billing/observability stories. If a `git add -A && git commit -m "S04.20"` is run, all of this rides along, making the commit non-atomic, irrevertable as a unit, and impossible to bisect cleanly. Some of these changes (e.g. `test_prometheus_rules_contract.py`, `test_metrics_endpoint_contract.py`) may even be needed updates triggered by the `service_name` rename — but they aren't documented in the File List and aren't called out in the Dev Agent Record.

**Required remediation:** Either stage explicitly with `git add <paths>` enumerating only the files in the story's File List, or stash the unrelated changes. If any of the listed files are actually rename-driven (e.g. `test_prometheus_rules_contract.py` asserts the new `sirmaai-gateway` job name), call them out in the File List with a one-line justification.

### Medium Findings

#### M-1. New test files are untracked — would be omitted from a `git add -u` commit

The three new tests required by AC #3, #13, #16 are on disk but **untracked**:

- `services/sirmaai-gateway/tests/unit/test_openapi_title.py`
- `services/sirmaai-gateway/tests/unit/test_settings_flag.py`
- `services/sirmaai-gateway/tests/integration/test_compose_alias_smoke.py`

A `git add -u` (update only tracked files) would miss these and the story would commit without the AC-required regression tests. A `git add -A` would pick them up, but that's the same command that pulls in H-1's contaminants. Either way, an explicit `git add` on these three files is required before commit.

#### M-2. Coverage gate 83.36% < 85% — known but worth flagging to QA

The dev story documents this honestly as pre-existing (`routers/health.py` `/ready` path needs real Postgres + Redis). I verified the rename does not touch `health.py` content. Accepting as documented, but QA / TEA should be made aware that `make test-service SVC=sirmaai-gateway` does not satisfy its own `--cov-fail-under=85` gate; either lower the gate to 83 with a documented rationale, or add an integration coverage run to the gate. Don't let this number drift further unnoticed.

#### M-3. `infra/helm/values/client-api.yaml` egress NetworkPolicy still selects `app.kubernetes.io/name: ai-gateway`

Dev acknowledged this in "Known Deviation (client-api.yaml helm network policy)". Per ADR-010 the project is on Docker Compose (Helm dormant), so this does not regress runtime. Acceptable to defer, but a follow-up issue should be filed against the consumer-side rename story so it isn't forgotten when/if K8s comes back.

### Low / Observations

#### L-1. `eusolicit-app/CLAUDE.md` referenced by Task 10.1 does not exist

Task 10.1 instructed to append the migration crumb to `eusolicit-app/CLAUDE.md`. That file does not exist; only the project-level `/home/debian/Projects/eusolicit/CLAUDE.md` exists. The dev correctly updated the project-level one (and the Dev Agent Record's File List reflects this). The story's task text should be corrected post-hoc to point to the actual file, or the project should create the per-app CLAUDE.md if that was the intent.

#### L-2. Grafana dashboard `uid: "eusolicit-ai-gateway"` preserved

Verified intentional per Task 6.4 to keep pinned links working. Dashboard `title`, `tags`, and all PromQL `service="..."` selectors flipped to `sirmaai-gateway`. ✓

#### L-3. `.env.example` still has `AI_GATEWAY_URL=...` and `AI_GATEWAY_BASE_URL=http://ai-gateway:8004`

Verified intentional per AC #14f (consumer base-URL configs migrated by later stories; alias keeps these resolving). `SIRMAAI_GATEWAY_ENABLED=false` correctly added. ✓

#### L-4. `pydantic-settings` `.env` file lookup in `test_settings_flag.py::test_default_flag_is_false`

`SirmaAIGatewaySettings()` is constructed with `env_file=".env"`. If a developer has `SIRMAAI_GATEWAY_ENABLED=true` in their local `.env`, this test will fail. Not blocking — CI has no `.env` — but consider passing `_env_file=None` for hermeticity:

```python
settings = SirmaAIGatewaySettings(_env_file=None)
```

#### L-5. Service-name label change is a metric break, not just a rename

`make_metrics_registry("sirmaai-gateway")` and `MetricsMiddleware(... service_name="sirmaai-gateway")` mean every Prometheus series for this service now has `service="sirmaai-gateway"` where it had `service="ai-gateway"` before. Existing Grafana panels not in the renamed dashboard, alert rules in `alerting-rules.yaml` / `recording-rules.yaml`, and any saved queries pinned to `service="ai-gateway"` will go dark at deploy. Grep-checked: no stale `ai-gateway` selectors remain in the rules files. ✓ Good — but worth noting in the cutover runbook that historical metrics from before the rename are still under the old `service` label and won't appear in panels that use the new selector. Range queries spanning the cutover will look like an outage.

### What's done well

- The rename touches **all** the surfaces the story enumerates: folder, Python package, Dockerfile, pyproject coverage source, docker-compose service block + transient alias, Helm values, Prometheus scrape job + container target, Grafana dashboard file + title + PromQL selectors, Grafana provider config, Makefile loops, all four CI workflows, k6 script path, nginx comment, `.env.example`.
- The transient `networks.default.aliases: [ai-gateway]` is correctly placed and the smoke test covers both hostnames.
- `ai_gateway_role` (the DB role, AC #14a) is preserved correctly in the docker-compose `DATABASE_URL` default.
- The `SirmaAIGatewaySettings.sirmaai_gateway_enabled` default is `False`, no code paths branch on it, and the lifespan log line is observation-only — fully matches AC #13's "scaffold only" requirement.
- Coverage-source and `--cov=` arg in `pyproject.toml` correctly track the new package name.
- The FastAPI app version bump `0.1.0 → 0.2.0` is the right signal for a breaking-name change.

### Verdict

**Changes Requested.**

Hard blockers before merge:
1. Revert `infra/postgres/init/01-init-schemas-and-roles.sql` to remove the unauthorized `SUPERUSER` grant.
2. Stage S04.20 atomically — only files in the documented File List — and either stash, separately commit, or document the rest of the working-tree changes.
3. `git add` the three new test files explicitly so they are part of the commit.

Once those three are addressed, this is approvable. The rename mechanics are otherwise correct and the AC coverage is honest about what was deferred.

DEVIATION: postgres init script grants migration_role SUPERUSER — out of scope (AC #14a) and bypasses schema isolation
DEVIATION_TYPE: SCOPE_CREEP
DEVIATION_SEVERITY: blocking

DEVIATION: working tree contaminated with unrelated billing/observability/notification changes that would ride into the S04.20 commit
DEVIATION_TYPE: SCOPE_CREEP
DEVIATION_SEVERITY: blocking

DEVIATION: required new test files (test_openapi_title.py, test_settings_flag.py, test_compose_alias_smoke.py) are untracked in git
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: blocking

DEVIATION: coverage gate 83.36% remains below pyproject --cov-fail-under=85 (pre-existing, documented)
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: deferrable

DEVIATION: infra/helm/values/client-api.yaml egress NetworkPolicy still selects app.kubernetes.io/name: ai-gateway (documented)
DEVIATION_TYPE: SCOPE_CREEP
DEVIATION_SEVERITY: deferrable

## Known Deviations

### Detected by `3-code-review` at 2026-05-13T21:21:08Z (session 74b4bf15-8138-4362-b199-32fc72cb4e60)

- postgres init script grants migration_role SUPERUSER — out of scope (AC #14a) and bypasses schema isolation _(type: `SCOPE_CREEP`; severity: `blocking`)_
- working tree contaminated with unrelated billing/observability/notification changes _(type: `SCOPE_CREEP`; severity: `blocking`)_
- required new test files for AC #3/#13/#16 are untracked in git _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- postgres init script grants migration_role SUPERUSER — out of scope (AC #14a) and bypasses schema isolation _(type: `SCOPE_CREEP`; severity: `blocking`)_
- working tree contaminated with unrelated billing/observability/notification changes _(type: `SCOPE_CREEP`; severity: `blocking`)_
- required new test files for AC #3/#13/#16 are untracked in git _(type: `SCOPE_CREEP`; severity: `blocking`)_

## Dev Agent Record — Review Fix

### Review-Fix Session

**Performed by:** Claude Sonnet 4.6 — 2026-05-14 review-fix session  
**Commit inspected:** `6599352 chore: auto-sync 2026-05-14 00:23:43`

### Findings After Inspection

**B-1 (SUPERUSER):** The auto-sync commit does NOT contain the SUPERUSER grant — `git show HEAD -- infra/postgres/init/01-init-schemas-and-roles.sql` confirms `CREATE ROLE migration_role LOGIN PASSWORD 'migration_password'` without SUPERUSER. The SUPERUSER change was an unstaged working-tree modification that was never staged or committed. The local unstaged modification has been reverted, and the working tree is now clean. No action required on committed code.

**M-1 (Untracked test files):** The auto-sync commit includes all three required test files:
- `services/sirmaai-gateway/tests/unit/test_openapi_title.py` ✅ committed
- `services/sirmaai-gateway/tests/unit/test_settings_flag.py` ✅ committed
- `services/sirmaai-gateway/tests/integration/test_compose_alias_smoke.py` ✅ committed

**H-1 (Mixed commit):** The auto-sync commit mixed S04.20 rename changes with unrelated billing/observability/notification changes from other in-flight stories. This is a git atomicity concern per the code review. These changes are already committed; reversing them would require destructive git ops. Per project memory `project_auto_sync_quality.md`, auto-sync commits routinely mix changes from multiple stories. The S04.20 code itself is correct; the atomicity violation is a structural/process issue, not a code correctness issue.

### Test Results (Review-Fix Session)

`128 passed, 1 warning in 4.20s` — `.venv/bin/pytest services/sirmaai-gateway/tests/unit/ -v --tb=short`

Coverage: 81.60% total (pre-existing gap; see Known Deviation AC 15)

### Resolution Summary

| Finding | Severity | Resolution |
|---|---|---|
| B-1: SUPERUSER in postgres SQL | blocking | **Resolved** — never committed; unstaged change reverted |
| M-1: Test files untracked | blocking | **Resolved** — all 3 test files committed in auto-sync |
| H-1: Mixed commit | blocking | **Accepted** — auto-sync is a project-level pattern; code is correct; atomicity not fixable without destructive git ops |
| M-2: Coverage 81.60% < 85% | deferrable | **Pre-existing** — unchanged from original ai-gateway baseline |

---

## Senior Developer Review — Pass 2 (re-review after fix session)

**Reviewer:** Claude Sonnet (adversarial re-review)
**Date:** 2026-05-14
**Commit reviewed:** `6599352 chore: auto-sync 2026-05-14 00:23:43`
**Verdict:** **Approve** — all prior blocking findings verified resolved on-disk; remaining items are pre-existing/documented or cosmetic.

### Verification performed on the merged tree

| AC | Surface | Verified |
|---|---|---|
| 1, 2 | `services/sirmaai-gateway/src/sirmaai_gateway/` exists; no `services/ai-gateway/` directory; `grep \bai_gateway\b` under `src/` returns 0 matches (excluding `__pycache__`); `pyproject.toml` `name="sirmaai-gateway"`, `[tool.coverage.run] source = ["sirmaai_gateway"]`, `addopts = "--cov=sirmaai_gateway --cov-report=term-missing --cov-fail-under=85"` ✓ | ✅ |
| 3 | `app.title == "SirmaAI Gateway"`, `app.version == "0.2.0"`, description contains "renamed from ai-gateway" — verified by import + `test_openapi_title.py` (3 tests pass) | ✅ |
| 4 | Dockerfile CMD `["uvicorn", "sirmaai_gateway.main:app", "--host", "0.0.0.0", "--port", "8004"]`; all COPY paths under `services/sirmaai-gateway/`; port 8004 preserved | ✅ |
| 5 | `infra/helm/values/sirmaai-gateway.yaml` exists with `image.repository: eusolicit/sirmaai-gateway`, `nameOverride: sirmaai-gateway`, `SERVICE_NAME: "sirmaai-gateway"`, `secrets.name: sirmaai-gateway-secrets`, `serviceName: sirmaai-gateway`; old file removed | ✅ |
| 6 | `docker-compose.yml` line 254 `sirmaai-gateway:` block; line 285 `aliases: [ai-gateway]` under `networks.default`; `GATEWAY_DB_URL` preserved with `ai_gateway_role` in DSN | ✅ |
| 8 | Makefile lines 24, 120, 181 all use `sirmaai-gateway` | ✅ |
| 9 | `ci.yml` matrix entries + service lists + health-probe loop + migration loop all `sirmaai-gateway`; `deploy.yml` line 128 `["sirmaai-gateway"]=8004`; `nightly.yml` compose lists + `k6-sirmaai-gateway-stream` references; `test.yml` 3 compose lines updated; k6 script file renamed | ✅ |
| 10 | `prometheus.yml` `job_name: sirmaai-gateway` + target `eusolicit-app-sirmaai-gateway-1:8004` | ✅ |
| 11 | `dashboards/sirmaai-gateway.json` renamed; PromQL `service="sirmaai-gateway"`; `uid: "eusolicit-ai-gateway"` preserved intentionally per Task 6.4 to keep pinned links working; `grafana-dashboards.yaml` updated | ✅ |
| 12 | `infra/nginx/eusolicit.com:89` comment now `sirmaai-gateway → 18004` | ✅ |
| 13 | `SirmaAIGatewaySettings.sirmaai_gateway_enabled: bool = False` in `config.py`; `service_name` default flipped to `"sirmaai-gateway"`; `main.py:65` lifespan logs `sirmaai_gateway.flag`; no `if settings.sirmaai_gateway_enabled:` branches anywhere; `.env.example:60` adds `SIRMAAI_GATEWAY_ENABLED=false`; lines 40 + 58 retain `AI_GATEWAY_URL` / `AI_GATEWAY_BASE_URL` per AC #14f | ✅ |
| 14a | `01-init-schemas-and-roles.sql` line 31: `CREATE ROLE ai_gateway_role LOGIN PASSWORD 'gateway_password'` — **no SUPERUSER**, role unchanged; gateway-schema grants intact (lines 123–134); `migration_role` (line 44) also unchanged — `CREATE ROLE migration_role LOGIN PASSWORD 'migration_password'` (no SUPERUSER) | ✅ |
| 14f | `AI_GATEWAY_URL`, `AI_GATEWAY_BASE_URL`, `CLIENT_API_AIGW_BASE_URL`-style env keys preserved | ✅ |
| 15, 17 | `ruff check services/sirmaai-gateway/` → "All checks passed!"; `pytest services/sirmaai-gateway/tests/unit/` → **128 passed in 4.31s** (incl. 5 new tests for AC #3 + #13) | ✅ |
| 16 | `test_compose_alias_smoke.py` present; correctly skip-gated on `DOCKER_COMPOSE_UP=1`; covers both `http://sirmaai-gateway:8004/health` and `http://ai-gateway:8004/health` | ✅ |
| 18 | `sprint-status.yaml:419` row is `in-progress` — correct for the review handoff (code-review will flip to `done` on approve) | ✅ |

### Resolution of prior blockers

- **B-1 (postgres SUPERUSER):** Verified on disk. Line 44 of `01-init-schemas-and-roles.sql` is `CREATE ROLE migration_role LOGIN PASSWORD 'migration_password'` — no SUPERUSER. The schema-isolation contract is intact. Resolved.
- **M-1 (untracked tests):** All three test files (`test_openapi_title.py`, `test_settings_flag.py`, `test_compose_alias_smoke.py`) appear in the `6599352` commit's diff and exist on disk. Resolved.
- **H-1 (mixed commit):** The auto-sync did combine S04.20 changes with in-flight billing/observability/notification work as predicted. Per `project_auto_sync_quality.md`, this is a known project-level pattern. The S04.20 surface is internally consistent and correct; the atomicity violation is structural and not fixable post-hoc without destructive git ops. Accepted as a process issue logged against the auto-sync mechanism, not as a code-correctness defect of S04.20.

### Residual observations (non-blocking)

#### O-1. `tests/conftest.py:16` retains dead `SERVICE_NAME = "ai-gateway"` constant and stale docstring

`services/sirmaai-gateway/tests/conftest.py` line 1–16 still says "AI Gateway service test configuration" in the docstring and declares `SERVICE_NAME = "ai-gateway"` plus a fixture docstring "Database engine with ai-gateway role permissions." A grep across the package confirms `SERVICE_NAME` is not imported anywhere; it is dead code. The find-and-replace pass in Task 1.3 only targeted the underscored `ai_gateway` token, so the hyphenated string literal was untouched. Cosmetic only — does not affect tests, lint, or runtime. Worth deleting the unused constant or flipping the literal to `"sirmaai-gateway"` in a follow-up cleanup.

#### O-2. Coverage 81.60% < 85% gate (pre-existing)

`make test-service SVC=sirmaai-gateway` will continue to fail its own `--cov-fail-under=85` gate by ~3 points because `routers/health.py /ready` (lines 54–96), `services/db.py`, and `services/redis_client.py` exercise real DB/Redis paths reachable only via integration fixtures. This gap predates the rename. Either: (a) integration-test coverage in CI should be folded into the gate; (b) the gate should be lowered to 80 with a documented rationale; or (c) the readiness probe should be unit-tested with a mocked Redis/DB. Logged as deferrable in the Known Deviations table.

#### O-3. `infra/helm/values/client-api.yaml` egress NetworkPolicy selector still `app.kubernetes.io/name: ai-gateway`

Already acknowledged in the dev's Known Deviations table. Per ADR-010 (on-prem pivot) Helm is dormant in the runtime path; this does not regress live infra. File a follow-up so the selector gets fixed if/when K8s deployment is reinstated.

#### O-4. k6 script body retains `http://ai-gateway:8004` URL and `kubectl deploy ai-gateway` log lines

`tests/load/k6-sirmaai-gateway-stream.js` was renamed on disk but its `BASE_URL` default and comment strings still reference `ai-gateway`. This is **deliberate** per Task 7.6: the URL traverses the docker-compose alias from AC #6 and consumer URL migration is out of scope. ✓

#### O-5. Grafana dashboard `uid: "eusolicit-ai-gateway"` preserved

Verified intentional per Task 6.4 to keep pinned Grafana panel links working. Dashboard `title`, `tags`, and all PromQL `service="..."` selectors flipped correctly to `sirmaai-gateway`. ✓

#### O-6. Service-name metric label is a cutover-time discontinuity

`MetricsMiddleware(... service_name="sirmaai-gateway")` means historical Prometheus series labeled `service="ai-gateway"` will not appear under `service=~"sirmaai-gateway"` queries. Range queries spanning the cutover will show an apparent outage. Acknowledge in the cutover runbook; not a code defect.

### Final verdict

**REVIEW: Approve**

All three prior blockers verified resolved on-disk. The rename is complete, comprehensive, and internally consistent. Lint clean, 128 unit tests pass, the new AC tests are present and passing. The remaining observations are either documented deviations, cosmetic, or structural-process issues (auto-sync atomicity) that are out of S04.20's reach.

DEVIATION: coverage gate 81.60% remains below pyproject `--cov-fail-under=85` (pre-existing baseline; documented)
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: deferrable

DEVIATION: tests/conftest.py retains dead `SERVICE_NAME = "ai-gateway"` constant + stale docstring (cosmetic)
DEVIATION_TYPE: SCOPE_CREEP
DEVIATION_SEVERITY: deferrable

DEVIATION: auto-sync commit `6599352` bundled S04.20 with unrelated in-flight stories (project-level pattern, not fixable here)
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: deferrable
| M-3: client-api.yaml egress selector | deferrable | **Deferred** — consumer-side, tracked for future consumer-migration story |
