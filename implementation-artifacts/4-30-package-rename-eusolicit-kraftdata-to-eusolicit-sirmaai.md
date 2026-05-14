# Story 4.30: Package Rename — `eusolicit-kraftdata` → `eusolicit-sirmaai`

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer landing the package-level half of the SirmaAI rebrand inside Epic 4 (Concern #3 from `implementation-readiness-report-2026-05-12-sirmaai.md` line 220 — "The shared Python package `packages/eusolicit-kraftdata/` ships request/response DTOs whose names still announce 'KraftData' even though the upstream platform is now SirmaAI; every consuming service carries `from eusolicit_kraftdata...` imports that read as legacy branding") + Epic 4 amendment line 512 ("**S04.30 Package rename — `eusolicit-kraftdata` → `eusolicit-sirmaai`** … Rename `packages/eusolicit-kraftdata/` → `packages/eusolicit-sirmaai/`. Update `pyproject.toml` package name. Regenerate typed client from `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json`. Update `pyproject.toml` dependency in `services/client-api/`, `services/admin-api/`, `services/data-pipeline/`. Find-and-replace import statements across all services. Backward-compatibility shim retired after 1 sprint (no historic Python clients to support — internal-only package). Type-check green across all consuming services. Closes readiness Concern #3.")**,

I want **a single mechanical-but-thorough pass that (a) renames the on-disk directory `eusolicit-app/packages/eusolicit-kraftdata/` → `eusolicit-app/packages/eusolicit-sirmaai/` via `git mv` (preserving every file's history); (b) renames the inner Python package directory `src/eusolicit_kraftdata/` → `src/eusolicit_sirmaai/` (the import name — `kraftdata` → `sirmaai` — case-sensitive, with no underscore variant); (c) updates the renamed package's `pyproject.toml` `[project] name = "eusolicit-kraftdata"` → `name = "eusolicit-sirmaai"` and bumps `version = "0.1.0"` → `version = "0.2.0"` (marks the rename as a breaking name change, mirroring the S04.20 `ai-gateway`→`sirmaai-gateway` version bump precedent); (d) regenerates the typed DTO surface against the SirmaAI OpenAPI spec at `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json` — pragmatic scope: align the existing `AgentRunRequest` / `WorkflowRunRequest` / `TeamRunRequest` / `StorageResourceRequest` / `WebhookRegistrationRequest` request DTOs and the `AgentRunResponse` / `WorkflowRunResponse` / `TeamRunResponse` / `StorageResourceResponse` / `WebhookEvent` / `StreamChunk` response DTOs and the `RunStatus` / `ResourceType` / `WebhookEventType` enums with the corresponding SirmaAI schema component names (`AgentRunRequestDto`, `RunResponseDto`, `WebhookEventDto`, etc. — the literal names exported by the OpenAPI spec's `components.schemas`), where field names diverge note the SirmaAI canonical name in a model docstring and keep the existing Python field name as the surface (consumers in `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` lines 43-49 + `services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` line 56 currently use the existing names and are NOT touched by this story beyond the import-statement substitution); the regeneration is **manual hand-curation against the spec** (NOT a `datamodel-code-generator` automated pass — the spec is 493 KB and exposes 200+ schema components, the vast majority of which the gateway does not consume and which would balloon the package surface area for zero consumer benefit; the canonical guidance in epic line 512 reads "Regenerate typed client" in the looser "align the typed DTOs with the spec" sense, NOT a literal mechanical codegen — the dev pass MUST document which spec components were inspected and how each existing DTO maps onto them in Completion Notes); (e) updates every `pyproject.toml` consumer dependency entry from `"eusolicit-kraftdata"` → `"eusolicit-sirmaai"` across the six services that list it (`services/client-api/`, `services/admin-api/`, `services/data-pipeline/`, `services/ai-gateway/` legacy folder still present, `services/sirmaai-gateway/`, `services/notification/`) AND the monorepo workspace `eusolicit-app/pyproject.toml` lines 14 + 27 (the `[tool.setuptools.packages.find]` workspace ref + the in-file comment); (f) updates every `from eusolicit_kraftdata...` / `import eusolicit_kraftdata` import statement across the codebase — concretely: `services/ai-gateway/src/ai_gateway/routers/execution.py` lines 26-27 (the legacy service folder still callable behind `SIRMAAI_GATEWAY_ENABLED=false` per S04.20 cutover), `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` lines 43-44, `services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` line 56 (a `TYPE_CHECKING`-only import — must update inside the guarded block), and every test file in `tests/` that imports the module (12+ files identified by the dev pre-flight grep); (g) renames the four package-specific test files at `eusolicit-app/tests/unit/`: `test_eusolicit_kraftdata_no_http_deps.py` → `test_eusolicit_sirmaai_no_http_deps.py`, `test_eusolicit_kraftdata_responses.py` → `test_eusolicit_sirmaai_responses.py`, `test_eusolicit_kraftdata_requests.py` → `test_eusolicit_sirmaai_requests.py`, `test_eusolicit_kraftdata_expanded.py` → `test_eusolicit_sirmaai_expanded.py` (each via `git mv` then content edits — the file names are the addressable testid prefix the CI matrix filters on per AC line below); (h) updates the test-utility / scaffold-assertion test files `tests/unit/test_scaffold_configs.py` (line 51 map literal), `tests/integration/test_scaffold_integration.py` (line 28 + multiple `subprocess.run([sys.executable, "-c", "import eusolicit_kraftdata; ..."]...)` invocations), `tests/smoke/test_scaffold_story_1_1.py` (line 60 + 477-488 — the `test_service_can_import_eusolicit_kraftdata` method and its assert string), `tests/unit/test_shared_packages_integration.py` (all `from eusolicit_kraftdata import ...` + `import eusolicit_kraftdata` + `eusolicit_kraftdata.__name__` references — 30+ touchpoints), and `tests/unit/test_ci_workflow_structure.py` (the `test_matrix_includes_eusolicit_kraftdata` method at line 263 — rename the method AND the matrix-entry string it asserts on); (i) updates every Dockerfile that performs `pip install /build/packages/eusolicit-kraftdata` — concretely: `services/data-pipeline/Dockerfile` line 10, `services/notification/Dockerfile` line 10, `services/admin-api/Dockerfile` line 10, `services/ai-gateway/Dockerfile` line 10, `services/client-api/Dockerfile` line 10 (the legacy `ai-gateway` service still has a Dockerfile and is still being built per S04.20 cutover discipline; `sirmaai-gateway` does NOT yet have its own Dockerfile — it builds from `services/ai-gateway/Dockerfile` per `docker-compose.yml` line 257 with the volume-mount overlay redirecting source paths; THIS story does NOT change that build topology — that's S04.20-cutover-final scope); (j) updates `eusolicit-app/docker-compose.yml` — every one of the 9 volume-mount lines that reads `./packages/eusolicit-kraftdata/src/eusolicit_kraftdata:/app/packages/eusolicit-kraftdata/src/eusolicit_kraftdata` (the dev-overlay volume mounts that hot-reload the package across all 9 services — `client-api`, `admin-api`, `data-pipeline`, `ai-gateway`, `notification`, `integrations-api`, `sirmaai-gateway`, plus the Celery worker variants); each becomes `./packages/eusolicit-sirmaai/src/eusolicit_sirmaai:/app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai`; (k) updates `.github/workflows/ci.yml` lines 36, 40, 44, 48, 52 (the per-service `deps:` matrix entries that pip-install the package) + lines 69-72 (the standalone shared-package matrix row `- project: eusolicit-kraftdata / path: packages/eusolicit-kraftdata / test-filter: eusolicit_kraftdata`) + `.github/workflows/nightly.yml` line 109 (the explicit `pip install -e packages/eusolicit-kraftdata` in the agent-error-handling job setup); (l) preserves **byte-for-byte** the existing module behaviour — the renamed `eusolicit_sirmaai/{requests,responses,enums}.py` retain every existing class name, field name, docstring text (with `KraftData` substituted to `SirmaAI` in user-facing prose where it appears verbatim — e.g. `requests.py` lines 1-7 module docstring, `responses.py` lines 1-7 module docstring, `enums.py` lines 1-6 module docstring, and every per-class docstring that says "KraftData agent" or "KraftData webhook" gets the rebrand — the DOCSTRINGS are the only prose surface, the field names and class identifiers are stable); (m) **NO backward-compat shim** — per epic line 512 "Backward-compatibility shim retired after 1 sprint (no historic Python clients to support — internal-only package)" → this story does a HARD rename in one pass, no `import eusolicit_kraftdata` alias remains in tree, no deprecation warnings emitted; the cutover is atomic and the next CI run is the gate; (n) preserves the existing prod docker image build paths — the `services/ai-gateway/Dockerfile` rename is OUT of scope (the service folder rename was S04.20's atomic concern with the network alias bridge, the Dockerfile inside it is service-local naming that survives the package rename without touch); the new `eusolicit-sirmaai` package builds and installs cleanly into both the legacy `ai-gateway` and the renamed `sirmaai-gateway` runtime via the existing Dockerfile pattern (`pip install --no-cache-dir /build/packages/eusolicit-sirmaai`)**,

so that **(1) Concern #3 from `implementation-readiness-report-2026-05-12-sirmaai.md` (line 220 + Concern-summary at line 414 — "shared package retains KraftData branding inconsistent with the SirmaAI platform rebrand") flips from ❌ GAP to ✅ Covered, removing the last shared-package surface that still announces the retired KraftData branding to anyone reading `pyproject.toml` dependencies or import statements; (2) Epic 4 amendment line 512 becomes implementation rather than aspiration — every consuming service is now importing from `eusolicit_sirmaai` and the workspace is correctly labelled across all the surfaces enumerated in the amendment (consumer pyprojects, Dockerfiles, compose volumes, CI matrix); (3) the SirmaAI pivot's brand-coherence invariant (per memory `project_sirmaai_pivot_2026_05_12.md`: "pre-launch re-platform onto SirmaAI; E04/E05/E11/E17 to refactor, E24-E28 to inject; don't recommend changes to retired surfaces") gets its final shared-package surface aligned — the only KraftData strings that survive after this story are (a) historical implementation-artifact files (e.g. `4-7-kraftdata-webhook-receiver-and-redis-stream-publishing.md` — sealed historical record per memory `feedback_investigate_dont_quiz.md`, NOT rewritten by this story), (b) deferred internal-naming surfaces inside `services/sirmaai-gateway/src/sirmaai_gateway/services/kraftdata_client.py` / `kraftdata_resilient.py` and the `_resolve_kraftdata_id()` / `_handle_kraftdata_error()` helpers in `routers/execution.py` and `routers/webhooks.py` (service-internal symbol names, NOT shared-package surface — scheduled for a separate hardening story alongside the deferred `infra/observability/prometheus/rules/kraftdata-isolation-rules.yaml` file rename), and (c) the still-tactical `KRAFTDATA_BASE_URL` / `KRAFTDATA_API_KEY` env-var keys (S04.22 introduced `SIRMAAI_BASE_URL` + per-Project keys but did not retire the legacy env vars, which remain readable for cutover-flag-off fallback — separately scheduled retirement); (4) the regeneration-against-OpenAPI-spec discipline (epic line 512) is honoured pragmatically — the dev pass reads the SirmaAI OpenAPI spec at `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json`, identifies the SirmaAI schema components that correspond to each existing DTO (e.g. SirmaAI's `AgentRunRequestDto` schema component matches our `AgentRunRequest` model — fields `message`, `sessionId`, `userId`, `tag`, `includeMessages`, `toolContext` in camelCase upstream; the existing Python model uses snake_case `message`, `session_id`, `user_id`, `tag`, `include_messages`, `tool_context` which is the project-wide convention and is preserved), and either (a) adopts any new optional fields the upstream spec has added that the gateway might want to surface to consumers, or (b) flags drift in Completion Notes with a follow-up backlog item if the upstream spec has changed enough that the existing models are incomplete — the Completion Notes section becomes the spec-compliance record for the rename; (5) consumer code that imports the renamed package gets a stable import surface from day one — the four-test rename + the scaffold-test updates encode the structural assertion that "the workspace contains a package named `eusolicit-sirmaai` whose import name is `eusolicit_sirmaai`" so future scaffold checks land on the right name (`tests/unit/test_scaffold_configs.py` is the canonical structural test per Story 1-1 + 1-7 lineage); (6) the CI matrix at `.github/workflows/ci.yml` continues to run the package's own test suite — the matrix row `- project: eusolicit-kraftdata / path: packages/eusolicit-kraftdata / test-filter: eusolicit_kraftdata` becomes `- project: eusolicit-sirmaai / path: packages/eusolicit-sirmaai / test-filter: eusolicit_sirmaai` so the file-glob `test_eusolicit_sirmaai_*.py` resolves to the renamed test files and the per-package CI lane stays green (per the comment in ci.yml lines 113-118 the filter is a file-glob NOT a `-k` filter — collection only loads matching `test_*.py` files, so the rename of the four test files MUST land at the same time as the matrix entry, otherwise collection grabs zero files and the lane reports "no tests collected" which is a CI failure mode); (7) the workspace `[tool.coverage.run] source = ["services", "packages"]` config at `eusolicit-app/pyproject.toml` line 51 needs no update (it points at the directories not the package names — the renamed `packages/eusolicit-sirmaai/` directory is automatically picked up), preserving the 80% coverage gate without operator intervention; (8) the change set is the minimal sanctioned response to Concern #3 + epic line 512 — no schema changes, no env-var renames, no Docker image repo renames, no Prometheus/Grafana surface renames (those concerns live in other stories or are intentionally deferred); the rename is a single-commit landing (per the AP18-C2 atomic discipline pattern used by S04.27/S04.28/S04.29) so a rollback is a single `git revert` plus the operator-side `docker compose down && docker compose up -d --build` rebuild cycle if the change has been deployed; (9) the next consumer or third-party engineer reading `eusolicit-app/services/client-api/pyproject.toml` no longer sees a `eusolicit-kraftdata` dependency that contradicts the SirmaAI platform branding everywhere else in the codebase, closing the last on-disk "KraftData mention in a structural file" surface in a single coherent commit**.

## Acceptance Criteria

1. **Package directory renamed via `git mv`** — `eusolicit-app/packages/eusolicit-kraftdata/` → `eusolicit-app/packages/eusolicit-sirmaai/`:
   - Use `git mv` not `mv` so each file's history is preserved across the rename (`git log --follow` continues to work on every file inside).
   - The renamed directory contains EXACTLY the same file tree as before, modulo the inner-package directory rename in AC 2 and the build artefacts cleanup in AC 14.
   - Specifically the on-disk structure becomes:
     ```
     eusolicit-app/packages/eusolicit-sirmaai/
       pyproject.toml
       src/
         eusolicit_sirmaai/
           __init__.py
           enums.py
           requests.py
           responses.py
     ```
   - No files in `eusolicit-app/packages/eusolicit-kraftdata/` remain on disk after this story lands.

2. **Inner Python package directory renamed via `git mv`** — `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_kraftdata/` → `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/`:
   - This is a SECOND `git mv` step performed AFTER AC 1's outer rename (chained: outer first, inner second — the alternative single-step rename would force git to track an unrelated subdirectory rename and yields uglier history).
   - The import name shifts from `eusolicit_kraftdata` to `eusolicit_sirmaai` (case-sensitive — Python imports are case-sensitive; underscore separator preserved).
   - Inside the renamed `eusolicit_sirmaai/__init__.py`, the existing `from eusolicit_kraftdata.requests import ...` (line 7) becomes `from eusolicit_sirmaai.requests import ...`. Both the `from .` relative-import form AND the existing absolute-form references update — for symmetry the absolute reference at `__init__.py:7` should match the relative-style imports at lines 15-19 + 22-29, i.e. switch all module-internal imports to relative-style: `from .requests import ...`, `from .enums import ...`, `from .responses import ...`. This eliminates a pre-existing inconsistency and makes future renames trivial.
   - The `__all__` list at lines 31-49 is preserved verbatim (class names are unchanged).
   - `__version__ = "0.1.0"` at line 3 becomes `__version__ = "0.2.0"` to match AC 3's pyproject version bump.

3. **`pyproject.toml` package-identity fields updated** at `eusolicit-app/packages/eusolicit-sirmaai/pyproject.toml`:
   - `[project] name = "eusolicit-kraftdata"` → `name = "eusolicit-sirmaai"`.
   - `[project] version = "0.1.0"` → `version = "0.2.0"` (marks the rename as a breaking name change, mirroring the S04.20 `version = "0.2.0"` precedent for the `sirmaai-gateway` service rename — see story 4-20 AC 3 + Task 2.2).
   - `[tool.setuptools.packages.find] where = ["src"]` preserved verbatim (the rename of the inner directory doesn't change `find` semantics).
   - Optional: add a `description = "EU Solicit ↔ SirmaAI integration DTOs (renamed from eusolicit-kraftdata 2026-05-14)"` line if it doesn't already exist (the current pyproject has no `description` field — adding it makes the rename self-documenting for `pip show eusolicit-sirmaai` output; this is **recommended** not required).

4. **Module docstring prose rebrand** — all three module-level docstrings AND every per-class docstring inside the renamed package update KraftData branding to SirmaAI:
   - `src/eusolicit_sirmaai/requests.py` line 2 "KraftData API request models" → "SirmaAI API request models"; line 4 "every KraftData API endpoint" → "every SirmaAI API endpoint"; line 6 "``eusolicit-common`` dependency" preserved verbatim (this is correct as-is — the package deliberately has zero common deps).
   - `src/eusolicit_sirmaai/responses.py` line 2 "KraftData API response models" → "SirmaAI API response models"; line 4 "every KraftData API endpoint" → "every SirmaAI API endpoint".
   - `src/eusolicit_sirmaai/enums.py` line 2 "KraftData status enums" → "SirmaAI status enums".
   - Per-class docstrings inside `requests.py`:
     - `AgentRunRequest` (line 15) "Request body for running a KraftData agent" → "Request body for running a SirmaAI agent"; "Maps to ``POST /client/api/v1/agents/{agentId}/run``" (line 17) preserved verbatim (the upstream route shape didn't change — the SirmaAI OpenAPI spec at `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json` confirms the path `POST /client/api/v1/agents/{agentId}/run` is unchanged).
     - `WorkflowRunRequest` (line 28) — same rebrand.
     - `TeamRunRequest` (line 39) — same.
     - `StorageResourceRequest` (line 52) "Metadata for uploading a file to KraftData storage" → "Metadata for uploading a file to SirmaAI storage"; path preserved.
     - `WebhookRegistrationRequest` (line 63) "Register a webhook endpoint with KraftData" → "Register a webhook endpoint with SirmaAI"; "Configures the KraftData platform" → "Configures the SirmaAI platform".
   - Per-class docstrings inside `responses.py`:
     - All five response classes (`AgentRunResponse`, `WorkflowRunResponse`, `TeamRunResponse`, `StorageResourceResponse`) rebrand "KraftData" → "SirmaAI" in their docstrings.
     - `WebhookEvent` (line 71) "Inbound webhook payload from KraftData platform" → "Inbound webhook payload from SirmaAI platform"; "Received at ``POST /webhooks/kraftdata``" → "Received at ``POST /webhooks/sirmaai``" (the receiver path was already changed by S04.25; this docstring was stale — rebrand fixes it at the same time as the package rename, since `webhooks.py:150` in sirmaai-gateway routes `POST /webhooks/sirmaai` not `POST /webhooks/kraftdata` per S04.25 — verify against the current code before committing the docstring update).
     - `StreamChunk` (line 90) — preserve the SSE-streaming description.
   - Per-class docstrings inside `enums.py`:
     - `RunStatus` (line 12) "Execution status of a KraftData agent, workflow, or team run" → "Execution status of a SirmaAI agent, workflow, or team run".
     - `ResourceType` (line 21) "KraftData entity type for webhook events" → "SirmaAI entity type for webhook events".
     - `WebhookEventType` (line 30) "Event types delivered via KraftData webhook callbacks" → "Event types delivered via SirmaAI webhook callbacks".
   - **NO field-name changes, NO class-name changes, NO enum-value changes** in this story — the wire-format DTOs the gateway exchanges with SirmaAI are unchanged on the wire; only the human-facing prose updates. (Field rename to SirmaAI camelCase would be a separate non-trivial change cascading through `routers/execution.py` and the test fixtures — explicitly OUT of scope per the "minimal sanctioned response" framing.)

5. **OpenAPI spec alignment notes captured in Completion Notes** — the dev pass MUST inspect `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json` and document in the story's Completion Notes section a per-DTO mapping table proving each existing model corresponds to a SirmaAI OpenAPI schema component:
   - For each of the 11 model classes (5 requests + 5 responses + WebhookEvent + StreamChunk minus duplicates) and 3 enums, note:
     - The Python class name (e.g. `AgentRunRequest`).
     - The SirmaAI OpenAPI `components.schemas` name (e.g. `AgentRunRequestDto` — or "not found, hand-rolled" if no upstream schema corresponds; this is fine, document it).
     - Any **divergence** flagged: missing fields, type drift, optional/required mismatch. For each divergence, classify as (a) "drift — file follow-up story" if the upstream has added fields the gateway would benefit from surfacing, OR (b) "intentional simplification — gateway exposes a subset of fields by design".
   - This table IS the deliverable for epic line 512's "Regenerate typed client" clause. The Completion Notes table becomes the spec-compliance record.
   - The dev pass is NOT required to add any new fields in this story — surfacing them is a follow-up story. The audit is the deliverable.
   - If the upstream OpenAPI spec is materially divergent from the existing DTOs (e.g. a request field has been renamed or a response has new required fields), flag it as a HIGH-priority follow-up; do NOT silently patch without spec-confirmation, and do NOT change wire field names without operator sign-off.

6. **Consumer `pyproject.toml` dependency updates** — every consumer file changes `"eusolicit-kraftdata"` → `"eusolicit-sirmaai"` (no version pin — internal package, the workspace `[tool.setuptools.packages.find]` resolves the source):
   - `services/client-api/pyproject.toml` line 30.
   - `services/admin-api/pyproject.toml` line 23.
   - `services/data-pipeline/pyproject.toml` line 21.
   - `services/ai-gateway/pyproject.toml` line 25 (the legacy service folder is still being built per S04.20 cutover discipline; updating its pyproject keeps the legacy CI lane green during the flag-off cutover window).
   - `services/sirmaai-gateway/pyproject.toml` line 25.
   - `services/notification/pyproject.toml` line 29.

7. **Monorepo workspace updates** at `eusolicit-app/pyproject.toml`:
   - Line 14 comment `# - packages/eusolicit-kraftdata` → `# - packages/eusolicit-sirmaai`.
   - Line 27 in `[tool.setuptools.packages.find] where = [...]` array: `"packages/eusolicit-kraftdata/src"` → `"packages/eusolicit-sirmaai/src"`.
   - No other line in the workspace pyproject.toml changes.

8. **Dockerfile updates** — every Dockerfile that performs `pip install /build/packages/eusolicit-kraftdata` updates the path to `/build/packages/eusolicit-sirmaai`:
   - `services/data-pipeline/Dockerfile` line 10 (note: the surrounding `RUN pip install --no-cache-dir \\ /build/packages/eusolicit-common \\ /build/packages/eusolicit-models \\ /build/packages/eusolicit-kraftdata` continues to install all three shared packages in dependency-stable order; the rename swaps the last path).
   - `services/notification/Dockerfile` line 10.
   - `services/admin-api/Dockerfile` line 10.
   - `services/ai-gateway/Dockerfile` line 10 (legacy service Dockerfile still used by the legacy folder build per AC 6).
   - `services/client-api/Dockerfile` line 10.
   - **NOT** `services/sirmaai-gateway/Dockerfile` — that file does not exist (the sirmaai-gateway service builds from `services/ai-gateway/Dockerfile` per `docker-compose.yml` line 257 via the source-overlay convention — S04.20 cutover discipline). The sirmaai-gateway service's package install therefore goes through the legacy ai-gateway Dockerfile, which IS updated by this AC.
   - **NOT** `services/integrations-api/Dockerfile` — integrations-api does not depend on the package (its pyproject doesn't list `eusolicit-kraftdata`, confirmed by `.github/workflows/ci.yml` line 56 which omits the dep). Verify before editing — if the file does happen to install the package, update it; if not, document the omission in Completion Notes.

9. **Docker-compose volume mount updates** at `eusolicit-app/docker-compose.yml`:
   - Every line of the form `./packages/eusolicit-kraftdata/src/eusolicit_kraftdata:/app/packages/eusolicit-kraftdata/src/eusolicit_kraftdata` becomes `./packages/eusolicit-sirmaai/src/eusolicit_sirmaai:/app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai`.
   - Per pre-flight grep there are **9** such lines (one per service block: client-api, admin-api, data-pipeline, ai-gateway, sirmaai-gateway, notification, integrations-api if present, and the two Celery worker variants for data-pipeline + notification). Verify the count matches at edit time — if the count diverges, document it in Completion Notes.
   - **NOT** `docker-compose.prod.yml` — prod compose does NOT mount source volumes (containers run from baked-in image source, per the prod-discipline pattern). Confirm by grep before edit: `grep -n kraftdata docker-compose.prod.yml` must return zero hits. If hits are returned, escalate before proceeding (it would indicate a pattern drift between dev and prod compose that this story does NOT scope to fix).

10. **CI workflow updates** — `.github/workflows/ci.yml` AND `.github/workflows/nightly.yml`:
    - `ci.yml` lines 36, 40, 44, 48, 52 (per-service `deps:` matrix entries that pip-install the three shared packages) — every `pip install -e packages/eusolicit-kraftdata` becomes `pip install -e packages/eusolicit-sirmaai`. The ordering and surrounding `&&` operators preserved verbatim.
    - `ci.yml` lines 69-72 — the standalone shared-package matrix row:
      ```yaml
      - project: eusolicit-kraftdata
        path: packages/eusolicit-kraftdata
        test-path: tests/unit
        test-filter: eusolicit_kraftdata
      ```
      becomes:
      ```yaml
      - project: eusolicit-sirmaai
        path: packages/eusolicit-sirmaai
        test-path: tests/unit
        test-filter: eusolicit_sirmaai
      ```
    - `nightly.yml` line 109 — `pip install -e packages/eusolicit-kraftdata` → `pip install -e packages/eusolicit-sirmaai`.
    - No other CI changes — `deploy.yml`, `test.yml`, `quality-gates.yml` do not contain `eusolicit-kraftdata` references (confirm via grep before editing; if any file does, update it).

11. **Test file renames via `git mv`** — four test files at `eusolicit-app/tests/unit/`:
    - `test_eusolicit_kraftdata_no_http_deps.py` → `test_eusolicit_sirmaai_no_http_deps.py`.
    - `test_eusolicit_kraftdata_responses.py` → `test_eusolicit_sirmaai_responses.py`.
    - `test_eusolicit_kraftdata_requests.py` → `test_eusolicit_sirmaai_requests.py`.
    - `test_eusolicit_kraftdata_expanded.py` → `test_eusolicit_sirmaai_expanded.py`.
    - Inside each renamed file, every `from eusolicit_kraftdata ... import ...` / `import eusolicit_kraftdata` / `eusolicit_kraftdata.` reference is updated to `eusolicit_sirmaai`. Per-test docstrings that mention "KraftData" prose are rebranded to "SirmaAI" symmetrically with AC 4 (the test prose is structural documentation for the package surface).
    - `test_eusolicit_sirmaai_no_http_deps.py` specifically — line 32 references the package path string `"packages/eusolicit-kraftdata/src/eusolicit_kraftdata"` (the AST-scan source path) which becomes `"packages/eusolicit-sirmaai/src/eusolicit_sirmaai"`.

12. **Scaffold-assertion test updates** — five test files outside the four renamed ones:
    - `tests/unit/test_scaffold_configs.py` line 51 map literal `"eusolicit-kraftdata": "eusolicit_kraftdata"` → `"eusolicit-sirmaai": "eusolicit_sirmaai"`.
    - `tests/smoke/test_scaffold_story_1_1.py`:
      - Line 60 map literal — same change as test_scaffold_configs.
      - Lines 477-488 — the method `test_service_can_import_eusolicit_kraftdata` is renamed to `test_service_can_import_eusolicit_sirmaai`; its inner subprocess call `[sys.executable, "-c", "import eusolicit_kraftdata; print(eusolicit_kraftdata.__version__)"]` updates to `import eusolicit_sirmaai; print(eusolicit_sirmaai.__version__)`; the assertion error string at line 488 `f"Failed to import eusolicit_kraftdata from services/{service}/: "` updates to `f"Failed to import eusolicit_sirmaai from services/{service}/: "`.
    - `tests/integration/test_scaffold_integration.py`:
      - Line 28 map literal — same change.
      - Lines 59, 62, 83, 86, 87, 106, 109, 162, 248, 249, 250, 274, 277 — every `eusolicit_kraftdata` token (inside subprocess command strings AND inside imports of import-cycle test fixtures) becomes `eusolicit_sirmaai`. Read each touch carefully — some are inside f-string subprocess command strings where the quoting matters.
    - `tests/unit/test_shared_packages_integration.py`:
      - Line 50 method `test_eusolicit_kraftdata_importable` → `test_eusolicit_sirmaai_importable` AND its docstring + body.
      - Every `from eusolicit_kraftdata import ...` (5+ occurrences) → `from eusolicit_sirmaai import ...`.
      - Every `eusolicit_kraftdata.__name__` / `eusolicit_kraftdata.__version__` reference → `eusolicit_sirmaai.__name__` / `eusolicit_sirmaai.__version__`.
      - The `importlib.reload(eusolicit_kraftdata)` call → `importlib.reload(eusolicit_sirmaai)`.
      - The class identity assertion line 65 `assert eusolicit_models.__name__ != eusolicit_kraftdata.__name__` (and twin at line 88) → swap to `eusolicit_sirmaai`.
      - Method `test_kraftdata_submodules_import_without_circular_dependency` (line 75) → `test_sirmaai_submodules_import_without_circular_dependency` AND its docstring/body.
      - Method `test_kraftdata_can_be_reloaded` (line 88) → `test_sirmaai_can_be_reloaded` AND body.
    - `tests/unit/test_ci_workflow_structure.py`:
      - Line 263 method `test_matrix_includes_eusolicit_kraftdata` → `test_matrix_includes_eusolicit_sirmaai`.
      - The assertion string inside the method (asserting the matrix entry contains the project name) updates from `"eusolicit-kraftdata"` to `"eusolicit-sirmaai"`.

13. **Service `import eusolicit_kraftdata` updates** — every service-level source file that imports the module:
    - `services/ai-gateway/src/ai_gateway/routers/execution.py` lines 26-27: `from eusolicit_kraftdata.requests import AgentRunRequest, TeamRunRequest, WorkflowRunRequest` → `from eusolicit_sirmaai.requests import AgentRunRequest, TeamRunRequest, WorkflowRunRequest`; same for the `from eusolicit_kraftdata.responses import (...)` block immediately below.
    - `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` lines 43-49 (or whatever the line numbers are at edit time — pre-flight grep before edit to confirm): same two-block update.
    - `services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` line 56 — INSIDE a `TYPE_CHECKING:` guarded block (line 55-60). The import line `from eusolicit_kraftdata.requests import AgentRunRequest` becomes `from eusolicit_sirmaai.requests import AgentRunRequest`. **CRITICAL**: do NOT move the import out of the TYPE_CHECKING block — the existing import is intentionally type-only, and moving it to module top would create a runtime cycle (the orchestrator constructs request DTOs lazily; verifying this is preserved is part of the dev pass's read-the-file-before-editing discipline per CLAUDE.md "READ FILES BEING MODIFIED" reminder).
    - **NOT** any of the `kraftdata_client.py` / `kraftdata_resilient.py` files in `services/sirmaai-gateway/src/sirmaai_gateway/services/` — those are service-LOCAL module names (the `kraftdata_client` inside sirmaai-gateway is a long-deprecated-name-not-yet-renamed local helper, not an import from the shared package). Renaming service-local modules is **out of scope** for this story per the "shared package" framing of epic line 512.
    - **NOT** any `_resolve_kraftdata_id()` / `_handle_kraftdata_error()` / `_resolve_kraftdata_via_agent_map()` helper symbols inside `routers/execution.py` and `routers/webhooks.py` — same reasoning as above; they are service-internal symbol names not imports from the shared package.

14. **Build-artefacts cleanup** — the existing `eusolicit-app/packages/eusolicit-kraftdata/build/` directory and `src/eusolicit_kraftdata.egg-info/` directory + any `__pycache__/` directories MUST NOT be carried over to the renamed package:
    - These directories are setuptools build artefacts and Python bytecode caches, not source. They were inadvertently committed at some point in the past (or are local-only — the dev pass should `git ls-files packages/eusolicit-kraftdata/build/` to determine which case applies).
    - If `git ls-files` returns the build artefacts as tracked, do `git rm -r packages/eusolicit-kraftdata/build/ packages/eusolicit-kraftdata/src/eusolicit_kraftdata.egg-info/` as a CLEAN-UP step BEFORE the `git mv` of the outer directory.
    - If they are untracked (in `.gitignore` for example), the `git mv` of the outer directory carries only the tracked files, so the artefacts naturally stay behind in the old (now-renamed) directory and need a separate `rm -rf eusolicit-app/packages/eusolicit-kraftdata/` cleanup after the `git mv` (the new directory is `eusolicit-sirmaai/`; the old name should not exist on disk at all after the rename).
    - **NEVER** carry the build artefacts forward into the renamed `packages/eusolicit-sirmaai/build/` — that would (a) leave stale bytecode under the old import name in the build directory, (b) bloat the rename diff unnecessarily, (c) drift from the rest of the workspace where `build/` and `*.egg-info/` are never tracked.
    - Document the cleanup decision in Completion Notes (tracked vs untracked + the corresponding command run).

15. **Cross-tenant negative test invariant** (project memory rule, explicitly called out even though this story is non-business-logic):
    - The shared package contains zero business-logic — only DTO schemas, enums, and module-level docstrings. There is no cross-tenant defense surface inside the package. The cross-tenant defense lives in the gateway service (S04.21 + S04.23 + S04.25 own the tenant-resolution paths).
    - This story does NOT add any new cross-tenant code paths and does NOT alter any existing ones. The rename is a pure naming change.
    - Document in Completion Notes: "AC 15 — no cross-tenant code paths touched; the package is DTO-only and the cross-tenant defense remains in S04.21 + S04.23 + S04.25 unit + integration test suites".

16. **No regression — every test suite stays green at the same level it was before this story**:
    - `make lint` clean across all services AND the workspace (ruff has no `eusolicit_kraftdata` references left to flag because the find-and-replace was exhaustive).
    - `make type-check` clean — mypy does not error on the renamed import name (the renamed package installs cleanly under the new name; the `from eusolicit_sirmaai...` imports resolve).
    - `make test-unit` clean — pure-logic tests don't touch the package at runtime except as imports.
    - `make test-service SVC=sirmaai-gateway` clean — the renamed package installs and the gateway's existing test suite continues to pass.
    - `make test-service SVC=ai-gateway` clean — the legacy ai-gateway service folder builds against the renamed package (its Dockerfile + pyproject got the rename per AC 6 + AC 8).
    - `make test-integration` clean — cross-service integration tests that depend on the gateway's DTO surface continue to pass; no flakiness.
    - The CI matrix lane `- project: eusolicit-sirmaai` (renamed per AC 10) shows green on the first post-rename CI run; the file-glob `test_eusolicit_sirmaai_*.py` resolves to the four renamed test files (per AC 11), and `pytest` reports 4 collected files + their tests passing.
    - **NO new failures introduced** — if ruff or mypy reports any error after this story's edits, the dev pass MUST fix the root cause before flipping the story to `review`. A stale `eusolicit_kraftdata` reference left behind is the most likely failure mode; the find-and-replace MUST be exhaustive.

17. **No backward-compatibility shim** — per epic line 512:
    - No `packages/eusolicit-kraftdata/` symlink to `packages/eusolicit-sirmaai/` after the rename.
    - No `eusolicit_kraftdata` shim module that re-exports from `eusolicit_sirmaai` with a deprecation warning.
    - The rename is **atomic and hard**. Anyone reading the codebase post-rename sees only `eusolicit-sirmaai` / `eusolicit_sirmaai` references; no historical alias remains. The implementation-readiness-report Concern #3 (line 220) framing explicitly anticipates this — the package is internal-only and there are no external Python clients that pin against `eusolicit-kraftdata`.

18. **DoD per project delivery rules** (the relevant subset — this is a structural rename, not a behavioural change):
    - `make lint` passes (`ruff check services/ packages/ tests/` clean).
    - `make type-check` passes (`mypy services/ packages/` clean) — note: pre-existing mypy errors in `sirmaai-gateway` flagged by Story 4.28 + 4.29 Review Pass-3 audit are NOT regressions of this story and should be documented as such in Completion Notes if they re-appear.
    - `make test-unit` passes.
    - `make test-integration` passes — assumes `make infra` running; if the dev pass runs in a host venv without postgres+redis, document the limitation per memory `project_test_execution_environment.md` and rely on CI as the gate.
    - `make coverage` passes (≥80% line coverage on touched modules — but note this rename is purely structural; coverage on `packages/eusolicit-sirmaai/` should be identical to coverage on the pre-rename `packages/eusolicit-kraftdata/` because the source content is unchanged).
    - Frontend gates — N/A (no frontend code touched).
    - E2E gates — N/A (no user flow touched; package rename is internal).

19. **Files this story touches (definitive list)** — used by the dev pass as a checklist; any file outside this list must be a justified addition documented in Completion Notes:
    - `eusolicit-app/packages/eusolicit-sirmaai/pyproject.toml` (renamed + edited per AC 3).
    - `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/__init__.py` (renamed via inner `git mv` + content edits per AC 2).
    - `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/enums.py` (renamed + docstring rebrand per AC 4).
    - `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/requests.py` (renamed + docstring rebrand per AC 4).
    - `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/responses.py` (renamed + docstring rebrand per AC 4).
    - `eusolicit-app/services/client-api/pyproject.toml` (line 30 — AC 6).
    - `eusolicit-app/services/admin-api/pyproject.toml` (line 23 — AC 6).
    - `eusolicit-app/services/data-pipeline/pyproject.toml` (line 21 — AC 6).
    - `eusolicit-app/services/ai-gateway/pyproject.toml` (line 25 — AC 6, legacy service folder).
    - `eusolicit-app/services/sirmaai-gateway/pyproject.toml` (line 25 — AC 6).
    - `eusolicit-app/services/notification/pyproject.toml` (line 29 — AC 6).
    - `eusolicit-app/pyproject.toml` (lines 14, 27 — AC 7).
    - `eusolicit-app/services/client-api/Dockerfile` (line 10 — AC 8).
    - `eusolicit-app/services/admin-api/Dockerfile` (line 10 — AC 8).
    - `eusolicit-app/services/data-pipeline/Dockerfile` (line 10 — AC 8).
    - `eusolicit-app/services/ai-gateway/Dockerfile` (line 10 — AC 8, also the build path for sirmaai-gateway per docker-compose).
    - `eusolicit-app/services/notification/Dockerfile` (line 10 — AC 8).
    - `eusolicit-app/docker-compose.yml` (9 volume-mount lines — AC 9).
    - `eusolicit-app/.github/workflows/ci.yml` (lines 36, 40, 44, 48, 52, 69-72 — AC 10).
    - `eusolicit-app/.github/workflows/nightly.yml` (line 109 — AC 10).
    - `eusolicit-app/tests/unit/test_eusolicit_sirmaai_no_http_deps.py` (renamed + content — AC 11).
    - `eusolicit-app/tests/unit/test_eusolicit_sirmaai_responses.py` (renamed + content — AC 11).
    - `eusolicit-app/tests/unit/test_eusolicit_sirmaai_requests.py` (renamed + content — AC 11).
    - `eusolicit-app/tests/unit/test_eusolicit_sirmaai_expanded.py` (renamed + content — AC 11).
    - `eusolicit-app/tests/unit/test_scaffold_configs.py` (line 51 — AC 12).
    - `eusolicit-app/tests/smoke/test_scaffold_story_1_1.py` (lines 60, 477-488 — AC 12).
    - `eusolicit-app/tests/integration/test_scaffold_integration.py` (lines 28, 59, 62, 83, 86, 87, 106, 109, 162, 248, 249, 250, 274, 277 — AC 12).
    - `eusolicit-app/tests/unit/test_shared_packages_integration.py` (every `eusolicit_kraftdata` reference — AC 12).
    - `eusolicit-app/tests/unit/test_ci_workflow_structure.py` (line 263 method + body — AC 12).
    - `eusolicit-app/services/ai-gateway/src/ai_gateway/routers/execution.py` (lines 26-27 — AC 13).
    - `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` (lines 43-49 — AC 13).
    - `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` (line 56, inside TYPE_CHECKING block — AC 13).
    - `eusolicit-docs/implementation-artifacts/4-30-package-rename-eusolicit-kraftdata-to-eusolicit-sirmaai.md` (this story file).
    - `eusolicit-docs/implementation-artifacts/sprint-status.yaml` (single-row status flip backlog → ready-for-dev → review → done as the story progresses).

20. **Files this story does NOT touch (sanity list)** — explicit non-goals:
    - `eusolicit-app/infra/observability/prometheus/rules/kraftdata-isolation-rules.yaml` — this is an SLO-isolation rule for the AI provider (the external dependency); the file naming + the `KraftDataDependentHighBurnRate` alert name + the `slo_target="kraftdata-dependent"` label are observability-concern strings that survive the package rename. Renaming the rule file requires updating the `MetricsMiddleware` injection logic at `services/*/middleware/metrics.py` (the `_KRAFTDATA_PATH_PATTERNS` regex pattern) and the Grafana / Alertmanager downstream routing — that's a separate operational hardening story, NOT scoped to this package rename. Documented as Concern #DEF1 (deferred) in the SirmaAI pivot's deferred-hardening backlog.
    - `services/sirmaai-gateway/src/sirmaai_gateway/services/kraftdata_client.py` and `kraftdata_resilient.py` — service-internal module names. Same deferred-hardening framing.
    - `_resolve_kraftdata_id()` / `_handle_kraftdata_error()` / `kraftdata_id` field on `AgentEntry` etc. — service-internal symbol names inside `sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` and `routers/webhooks.py` and `services/agent_registry.py`. Same.
    - `KRAFTDATA_BASE_URL` / `KRAFTDATA_API_KEY` env vars — config keys still in active use as flag-off fallback per S04.22 cutover discipline; their retirement is its own story alongside the legacy ai-gateway code-path deletion (post-S04.30 cutover completion).
    - `eusolicit-docs/implementation-artifacts/1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md` and similar historical implementation artefacts — sealed historical records per memory `feedback_investigate_dont_quiz.md`; NOT rewritten by this story.
    - `eusolicit-docs/test-artifacts/atdd-checklist-1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md` — historical ATDD checklist; sealed record.
    - `docker-compose.prod.yml` — prod compose has no `eusolicit-kraftdata` references (verified by pre-flight grep at AC 9); if any pop up, escalate, do NOT silently edit.
    - `scripts/deploy.sh` — confirm via grep that the deploy script doesn't reference the package; if so, no edit. Per memory `project_deploy_nginx_manual.md` the deploy.sh is the canonical shipping script and changes to it require operator review.
    - `.env.example` / `.env.prod.example` — confirm no `KRAFTDATA_*` to `SIRMAAI_*` env-var rename is in scope here (S04.22 introduced new env vars but did not retire the legacy ones; retirement is its own story).
    - Any Alembic migration file — the package rename does not require schema changes.
    - Any frontend file under `frontend/` — the package is Python-only.

21. **Sprint-status orchestrator-managed surgical update**:
    - At story creation time (this dispatch), `sprint-status.yaml` row `4-30-package-rename-eusolicit-kraftdata-to-eusolicit-sirmaai: backlog` is flipped to `ready-for-dev` via a SURGICAL single-row edit + audit-trail header prepend (per memory `project_sprint_status_file.md` — file is orchestrator-managed, surgical edits only, no structural regeneration).
    - Subsequent flips owned by dev-story / validate-story / code-review per BMAD pipeline discipline (`ready-for-dev` → `in-progress` → `review` → `done`).

## Tasks / Subtasks

- [x] **Task 1: Pre-flight grep audit + build-artefact decision** (AC 14, 19, 20)
  - [x] Run `grep -rln 'eusolicit-kraftdata\|eusolicit_kraftdata' eusolicit-app/ eusolicit-docs/test-artifacts/ 2>/dev/null` to enumerate every reference. Confirm the touchpoint count matches the file list in AC 19 (rough expected count: ~25-30 distinct files outside the package itself). If the count diverges materially, document the additional files and flag any unexpected location.
  - [x] Run `git ls-files packages/eusolicit-kraftdata/build/ packages/eusolicit-kraftdata/src/eusolicit_kraftdata.egg-info/` to determine if the build artefacts are tracked. Document the result in Completion Notes (tracked vs untracked + the cleanup command run per AC 14).
  - [x] Run `grep -n eusolicit-kraftdata eusolicit-app/docker-compose.prod.yml` — expected zero hits (AC 9 guard). If hits exist, HALT and file an escalation.
  - [x] Run `grep -rln eusolicit-kraftdata eusolicit-app/scripts/` — expected zero hits in `deploy.sh` (AC 20 guard). Document result.

- [x] **Task 2: Read all UPDATE-target files before editing** (CLAUDE.md "READ FILES BEING MODIFIED" reminder)
  - [x] Read `services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` lines 45-65 in full to confirm the `TYPE_CHECKING:` guarded import structure before mutating line 56. Document the pre-edit state in Completion Notes (TYPE_CHECKING block contents).
  - [x] Read `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` lines 1-100 in full to map the existing import block + the `KraftData*` exception imports (which stay on `sirmaai_gateway.services.exceptions` — internal namespace, NOT shared package). Confirm only lines 43-49 are package-import lines; lines 60-62 (`KraftDataAPIError`, `KraftDataConnectionError`, `KraftDataTimeoutError`) are local exception class re-exports and stay untouched.
  - [x] Read `services/ai-gateway/src/ai_gateway/routers/execution.py` lines 1-50 to confirm the import block structure mirrors the sirmaai-gateway twin. The legacy ai-gateway service stays callable per S04.20 cutover; this story keeps it green.
  - [x] Read `tests/unit/test_shared_packages_integration.py` end-to-end to map every `eusolicit_kraftdata` reference (the file has the most density — 30+ touchpoints across multiple test methods).
  - [x] Read `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json` (selectively — focus on `components.schemas` section; use `jq '.components.schemas | keys'` to enumerate the schema component names without loading the full 493 KB file into model context). Capture the schema-component names that correspond to each of the 11 model classes in this package for the AC 5 mapping table.

- [x] **Task 3: Build-artefact cleanup** (AC 14)
  - [x] Per Task 1 finding: if `build/` and `egg-info/` are tracked, run `git rm -rf eusolicit-app/packages/eusolicit-kraftdata/build/ eusolicit-app/packages/eusolicit-kraftdata/src/eusolicit_kraftdata.egg-info/`. Otherwise the post-rename `rm -rf eusolicit-app/packages/eusolicit-kraftdata/` cleanup (Task 4 sub-step) sweeps them.

- [x] **Task 4: Package directory + inner-package rename** (AC 1, 2)
  - [x] `git mv eusolicit-app/packages/eusolicit-kraftdata eusolicit-app/packages/eusolicit-sirmaai` — outer directory rename.
  - [x] `git mv eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_kraftdata eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai` — inner package rename.
  - [x] Verify no orphaned files remain: `find eusolicit-app/packages/ -type d -name '*kraftdata*'` must return empty.
  - [x] Verify `git status` reports the moves as renames (`R` in the `git status` output) not delete+add — confirms history preservation.

- [x] **Task 5: Package pyproject + `__init__.py` content edits** (AC 2, 3)
  - [x] Edit `eusolicit-app/packages/eusolicit-sirmaai/pyproject.toml`:
    - `name = "eusolicit-kraftdata"` → `name = "eusolicit-sirmaai"`.
    - `version = "0.1.0"` → `version = "0.2.0"`.
    - Optionally add `description = "EU Solicit ↔ SirmaAI integration DTOs (renamed from eusolicit-kraftdata 2026-05-14)"` (recommended per AC 3).
  - [x] Edit `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/__init__.py`:
    - Line 1 docstring `"""EU Solicit — KraftData integration types."""` → `"""EU Solicit — SirmaAI integration types."""`.
    - Line 3 `__version__ = "0.1.0"` → `__version__ = "0.2.0"`.
    - Line 7 `from eusolicit_kraftdata.requests import (` → `from .requests import (` (switch to relative-import form per AC 2).
    - Lines 15-19 `from .enums import (` — already relative, no change.
    - Lines 22-29 `from .responses import (` — already relative, no change.
    - `__all__` list at lines 31-49 preserved verbatim.

- [x] **Task 6: Docstring rebrand inside the renamed package** (AC 4)
  - [x] Edit `src/eusolicit_sirmaai/requests.py`:
    - Module docstring lines 1-7: `KraftData` → `SirmaAI`.
    - Per-class docstrings: AgentRunRequest, WorkflowRunRequest, TeamRunRequest, StorageResourceRequest, WebhookRegistrationRequest — `KraftData` → `SirmaAI`.
    - Path strings inside docstrings (e.g. `POST /client/api/v1/agents/{agentId}/run`) preserved verbatim.
  - [x] Edit `src/eusolicit_sirmaai/responses.py`:
    - Module docstring + per-class docstrings: same pattern.
    - `WebhookEvent` docstring: `POST /webhooks/kraftdata` → `POST /webhooks/sirmaai` (S04.25 receiver path correction).
  - [x] Edit `src/eusolicit_sirmaai/enums.py`:
    - Module docstring + per-class docstrings: same pattern.
  - [x] Run `grep -n KraftData eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/` — expected zero hits (the find-and-replace must be exhaustive on the prose).

- [x] **Task 7: OpenAPI alignment audit** (AC 5)
  - [x] Using the `jq` enumeration from Task 2: build the DTO-to-schema-component mapping table. For each of the 11 model classes + 3 enums in the renamed package, record (Python class, SirmaAI OpenAPI component name, divergences noted).
  - [x] Document the table in Completion Notes verbatim.
  - [x] Flag any HIGH-priority divergences (e.g. upstream renamed `sessionId` to `session_uuid`, added required field, changed enum value set) as follow-up backlog items. Do NOT silently patch wire-name divergences in this story.

- [x] **Task 8: Consumer pyproject + workspace pyproject updates** (AC 6, 7)
  - [x] Update each of 6 consumer pyprojects: client-api, admin-api, data-pipeline, ai-gateway, sirmaai-gateway, notification — line containing `"eusolicit-kraftdata"` → `"eusolicit-sirmaai"`.
  - [x] Update `eusolicit-app/pyproject.toml` line 14 comment + line 27 array element.
  - [x] Run `grep -rn 'eusolicit-kraftdata' eusolicit-app/services/*/pyproject.toml eusolicit-app/pyproject.toml` — expected zero hits.

- [x] **Task 9: Dockerfile updates** (AC 8)
  - [x] Update each of 5 Dockerfiles: data-pipeline, notification, admin-api, ai-gateway, client-api — `pip install /build/packages/eusolicit-kraftdata` → `/build/packages/eusolicit-sirmaai`.
  - [x] Run `grep -rn 'eusolicit-kraftdata' eusolicit-app/services/*/Dockerfile` — expected zero hits.
  - [x] Verify `services/integrations-api/Dockerfile` does NOT reference the package (confirmed by Task 1 grep); if it does, update it.

- [x] **Task 10: docker-compose.yml updates** (AC 9)
  - [x] Replace all 9 volume-mount lines per AC 9.
  - [x] Run `grep -n eusolicit-kraftdata eusolicit-app/docker-compose.yml` — expected zero hits.

- [x] **Task 11: CI workflow updates** (AC 10)
  - [x] `.github/workflows/ci.yml`:
    - Lines 36, 40, 44, 48, 52 — `pip install -e packages/eusolicit-kraftdata` → `pip install -e packages/eusolicit-sirmaai`.
    - Lines 69-72 — matrix entry block: `project: eusolicit-kraftdata` → `eusolicit-sirmaai`, `path: packages/eusolicit-kraftdata` → `packages/eusolicit-sirmaai`, `test-filter: eusolicit_kraftdata` → `eusolicit_sirmaai`.
  - [x] `.github/workflows/nightly.yml` line 109 — `pip install -e packages/eusolicit-kraftdata` → `pip install -e packages/eusolicit-sirmaai`.
  - [x] Run `grep -rn 'eusolicit-kraftdata\|eusolicit_kraftdata' eusolicit-app/.github/` — expected zero hits.

- [x] **Task 12: Test file renames + content updates** (AC 11, 12)
  - [x] `git mv` each of 4 package-specific test files at `tests/unit/`: `test_eusolicit_kraftdata_*.py` → `test_eusolicit_sirmaai_*.py`.
  - [x] Inside each renamed file, find-and-replace `eusolicit_kraftdata` → `eusolicit_sirmaai` (case-sensitive, AST-relevant); docstrings prose `KraftData` → `SirmaAI`. The path string in `test_eusolicit_sirmaai_no_http_deps.py` line 32 updates per AC 11.
  - [x] Update scaffold-assertion tests per AC 12: `test_scaffold_configs.py`, `test_scaffold_story_1_1.py`, `test_scaffold_integration.py`, `test_shared_packages_integration.py`, `test_ci_workflow_structure.py`. Each gets every `eusolicit_kraftdata` token swapped to `eusolicit_sirmaai`, every `eusolicit-kraftdata` (dash form) swapped to `eusolicit-sirmaai`, and method names containing `kraftdata` renamed to `sirmaai`. Read the file before editing — some references are inside f-string subprocess command strings where shell-quote correctness matters.
  - [x] Run `grep -rn 'eusolicit_kraftdata\|eusolicit-kraftdata' eusolicit-app/tests/` — expected zero hits.

- [x] **Task 13: Service-level import updates** (AC 13)
  - [x] `services/ai-gateway/src/ai_gateway/routers/execution.py` lines 26-27 — `from eusolicit_kraftdata.{requests,responses}` → `from eusolicit_sirmaai.{requests,responses}`.
  - [x] `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` lines 43-49 — same.
  - [x] `services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` line 56 (inside `TYPE_CHECKING:` guard) — `from eusolicit_kraftdata.requests import AgentRunRequest` → `from eusolicit_sirmaai.requests import AgentRunRequest`. Verify the import stays inside the guard block — do NOT move it to module top.
  - [x] Run `grep -rn 'eusolicit_kraftdata' eusolicit-app/services/` — expected zero hits (after Task 12 + Task 13 + the package's own internal absolute-form import switched to relative-form per Task 5). The only intra-package references should be via relative-form `from .requests/...responses/...enums import ...`.

- [x] **Task 14: Final whole-tree grep audit** (AC 16, 19)
  - [x] `grep -rn 'eusolicit-kraftdata\|eusolicit_kraftdata' eusolicit-app/ 2>/dev/null` — expected zero hits, with the following EXPLICIT exceptions documented in Completion Notes:
    - `.venv/` / vendor directories (third-party packages that happen to have similar names — none expected, but document if found).
    - Any committed `*.lock` / `*.lockb` files — if they cache the old package name, document as deferred (lockfile regeneration is a separate step).
  - [x] `grep -rn 'eusolicit-kraftdata\|eusolicit_kraftdata' eusolicit-docs/implementation-artifacts/ 2>/dev/null` — expected: many hits (historical implementation-artifact files reference the old package by name; THESE ARE SEALED HISTORICAL RECORDS per memory `feedback_investigate_dont_quiz.md` + the project's "don't rewrite history" discipline). The grep is informational only; no edits to historical implementation-artifact files.
  - [x] `grep -rn 'KraftData' eusolicit-app/packages/eusolicit-sirmaai/` — expected zero hits (Task 6 rebrand was exhaustive on the package surface).

- [x] **Task 15: Quality gates** (AC 16, 18)
  - [x] From `eusolicit-app/`: `make lint` — clean. If ruff flags any stale `eusolicit_kraftdata` references, fix and re-run.
  - [x] `make type-check` — clean against the renamed import name. Pre-existing mypy errors in `sirmaai-gateway` Python files (per Story 4.28 / 4.29 review notes) that re-appear are NOT regressions of this story; document them as such in Completion Notes.
  - [x] If `make infra` is up locally: `make test-unit`, `make test-integration`, `make test-service SVC=sirmaai-gateway` — all green.
  - [x] If `make infra` is NOT up: document the limitation per memory `project_test_execution_environment.md`; CI is the canonical gate.
  - [x] Optionally: `python -c "import eusolicit_sirmaai; print(eusolicit_sirmaai.__version__)"` from `eusolicit-app/` after `pip install -e packages/eusolicit-sirmaai` — should print `0.2.0`.

- [x] **Task 16: Documentation crumb in `eusolicit-app/CLAUDE.md`** (informational, low-priority)
  - [x] Append a one-paragraph entry under the "Active Service Migrations" subsection of `eusolicit-app/CLAUDE.md` noting: "`eusolicit-kraftdata` shared package has been renamed to `eusolicit-sirmaai` (S04.30) — import name `eusolicit_sirmaai`, version 0.2.0. No backward-compatibility shim; consumers update imports in this same commit. Closes implementation-readiness Concern #3."
  - [x] **Skip this Task IF** appending to CLAUDE.md is contentious (the file is already long); the dev pass MAY defer this Task to a separate doc-housekeeping pass without affecting AC coverage. Document the decision in Completion Notes.

- [x] **Task 17: Story finalization** (AC 21)
  - [x] Update `eusolicit-docs/implementation-artifacts/sprint-status.yaml` row `4-30-package-rename-eusolicit-kraftdata-to-eusolicit-sirmaai: backlog → ready-for-dev` at story-creation time (this dispatch, already handled by the create-story workflow).
  - [x] Subsequent flips (`ready-for-dev → in-progress → review → done`) owned by dev-story / validate-story / code-review per BMAD pipeline discipline.

## Dev Notes

### Architectural context (read before editing anything)

This story closes **Concern #3** from the SirmaAI pivot's implementation-readiness report at `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-12-sirmaai.md`. The shared Python package `packages/eusolicit-kraftdata/` was created during Story 1.7 as the typed-DTO surface for the KraftData Agentic AI platform integration. With the SirmaAI rebrand (per ADR-018), every operational and runtime surface has been migrated — except this package, which still announces "KraftData" in its directory name, its `pyproject.toml` `[project] name` field, its inner module name (`eusolicit_kraftdata`), and the import statements in every consuming service.

The Epic 4 amendment line 512 reads: *"S04.30 Package rename — `eusolicit-kraftdata` → `eusolicit-sirmaai`. Rename `packages/eusolicit-kraftdata/` → `packages/eusolicit-sirmaai/`. Update `pyproject.toml` package name. Regenerate typed client from `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json`. Update `pyproject.toml` dependency in `services/client-api/`, `services/admin-api/`, `services/data-pipeline/`. Find-and-replace import statements across all services. Backward-compatibility shim retired after 1 sprint (no historic Python clients to support — internal-only package). Type-check green across all consuming services. Closes readiness Concern #3."*

The scope is bounded by what the epic enumerates — package surface, pyproject deps, imports. Service-internal naming surfaces (e.g. `kraftdata_client.py` modules inside `sirmaai-gateway/services/`, `_resolve_kraftdata_id()` helpers, the `kraftdata-isolation-rules.yaml` observability file, the `KRAFTDATA_BASE_URL` env-var) are EXPLICITLY out of scope and tracked in the deferred-hardening backlog. Renaming them here would (a) balloon the story scope, (b) introduce risk to active flag-off code paths during the S04.20 cutover window, (c) collide with separate stories that own those surfaces.

### Why a hard rename without a backward-compat shim

Epic line 512 spells this out: *"Backward-compatibility shim retired after 1 sprint (no historic Python clients to support — internal-only package)."* The package is an internal workspace member — there is no external pip install / no PyPI publication / no semver contract with downstream users. Every consumer is in this monorepo and is updated in the same commit as the rename. A shim that `from eusolicit_kraftdata import *` from `eusolicit_sirmaai` with a deprecation warning would (a) add an alias dual-name surface that lingers indefinitely (one-sprint retirements often slip), (b) confuse `mypy` and IDE auto-completion (which name is canonical?), (c) generate noise warnings that drown out actual deprecation signals. Hard rename in a single commit is the right primitive.

### Why version bump 0.1.0 → 0.2.0

Following the S04.20 precedent for the `ai-gateway` → `sirmaai-gateway` service rename (where Task 2.2 bumped `version` from `0.1.0` to `0.2.0` to mark the rename as a breaking surface-name change). Even though this package is internal-only with no semver contract, the bump:
1. Documents the rename moment in the package's own metadata (`pip show eusolicit-sirmaai` shows `Version: 0.2.0` immediately after the rename, vs `0.1.0` for the old name — operators reading the metadata can correlate).
2. Mirrors the service-rename precedent so future renames follow the same pattern.
3. Has zero functional impact (no downstream pin checks the version).

### Why regenerate from OpenAPI is interpreted as "audit + map", not literal codegen

The SirmaAI OpenAPI spec at `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json` is 493 KB and exposes 200+ schema components covering the entire SirmaAI platform — Projects v1, Storage Resources v1, Agent Sessions, Workflow Sessions, Team Sessions, Traces, Compliance Frameworks, Stripe webhooks, Policy Test, Role Presets, Organization Administration, etc. The gateway service consumes a tiny fraction of this surface (the agent / workflow / team run endpoints + the webhook event types + the storage upload endpoint). A literal `datamodel-code-generator` pass would:
1. Generate 200+ Pydantic models, the vast majority of which the gateway never references.
2. Balloon the package's exported surface area, making accidental over-coupling more likely (any consumer can grab any DTO without restraint).
3. Force decisions about model naming (`AgentRunRequestDto` vs `AgentRunRequest`) that the existing thin hand-rolled approach has already settled.

The pragmatic interpretation — and the canonical guidance for this story — is: **audit the existing DTOs against the SirmaAI OpenAPI spec, document the mapping, flag drift as follow-up backlog items**. The audit table in Completion Notes per AC 5 IS the regeneration deliverable. If the upstream spec has materially diverged from the existing models (e.g. SirmaAI added a required `traceId` field to `AgentRunRequestDto`), that becomes a HIGH-priority follow-up — not a silent patch in this story.

### Why the test-file rename (AC 11) is not optional

The CI matrix at `.github/workflows/ci.yml` lines 113-124 uses a **file-glob** test filter (NOT a `-k` filter), specifically because `-k` would force `pytest` to import all 25 `test_*.py` files during collection, many of which import heavy deps absent from the shared-package CI env. The file-glob `test_${{ matrix.test-filter }}_*.py` resolves to a precise set of files for collection. The matrix entry for the renamed package becomes `test-filter: eusolicit_sirmaai`, which means `pytest` collects `test_eusolicit_sirmaai_*.py` files. If the four existing test files were not renamed, the collection would return zero files and the CI lane would report "no tests collected" — which is a CI failure mode. AC 11 + AC 10 are paired; both must land in the same commit.

### Why we keep snake_case field names (e.g. `session_id`)

The SirmaAI OpenAPI spec ships camelCase field names in JSON payloads (`sessionId`, `userId`, `includeMessages`). The existing Pydantic models in this package use snake_case attribute names (`session_id`, `user_id`, `include_messages`) WITHOUT pydantic field aliases — meaning serialization out of these models would produce snake_case JSON, which would NOT match the SirmaAI API's expected camelCase keys. This is a pre-existing latent defect in the package that this story does NOT fix, because:
1. The gateway-internal HTTP plumbing in `services/sirmaai-gateway/src/sirmaai_gateway/services/kraftdata_client.py` already serializes to camelCase (or at least matches the SirmaAI contract) — the request DTOs in this package are used as type hints for the inbound request body the gateway receives from EU Solicit's own services, not as serializers for the outbound request to SirmaAI. The internal-vs-external naming asymmetry is intentional.
2. Fixing this would require adding `Field(alias="sessionId")` on every field, switching to `ConfigDict(populate_by_name=True)`, and verifying every test passes — substantial scope creep.
3. The pre-existing pattern is consistent across all hand-rolled DTOs in the workspace (per `eusolicit-models` and other shared packages).

If the gateway team decides to migrate to camelCase aliases system-wide, that becomes a separate refactor story. This story stays scoped to the rename.

### Why we update the legacy `ai-gateway` folder's pyproject + Dockerfile + execution.py

S04.20 introduced the `sirmaai-gateway` service folder and added a docker-compose network alias `ai-gateway` for backward compatibility during the cutover. The legacy `services/ai-gateway/` folder is still on disk + still being built by CI + still callable when `SIRMAAI_GATEWAY_ENABLED=false`. The retirement of the legacy folder is a separate post-cutover story (not yet scheduled — happens after the prod flag flip per epic line 528-529 Phase 2). During the cutover window, the legacy folder MUST keep building and its imports MUST resolve. If we updated the `eusolicit-sirmaai` package but left `services/ai-gateway/src/ai_gateway/routers/execution.py` lines 26-27 importing `eusolicit_kraftdata`, the legacy folder would fail to import on its next build, breaking the cutover-flag-off fallback. So this story updates the legacy folder symmetrically with the renamed folder — both flag-on and flag-off code paths remain green.

### Why the `TYPE_CHECKING` import in `async_run_orchestrator.py` stays guarded

The import `from eusolicit_kraftdata.requests import AgentRunRequest` at line 56 of `services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` is inside an `if TYPE_CHECKING:` block (lines 55-60). This is intentional — the orchestrator constructs request DTOs lazily (via a factory or via passing already-constructed objects from the router), and a runtime import would create a cycle (`async_run_orchestrator` is imported by `routers/execution.py` which already imports `eusolicit_kraftdata.requests`). The TYPE_CHECKING guard breaks the cycle. AC 13 + Task 13 must preserve this — the import line moves inside the block, not outside.

### Why we don't rename the `kraftdata_client` / `kraftdata_resilient` service-local modules

These are internal naming surfaces inside `services/sirmaai-gateway/src/sirmaai_gateway/services/`. They were created during S04.01 / S04.02 when the gateway was still called `ai-gateway` and the upstream was still called KraftData. They have not been renamed yet because:
1. S04.20 (service rename) scoped the rename to the FOLDER + the PACKAGE + the SERVICE NAME, not internal module names. The cutover-discipline pattern is "minimum sanctioned change per story" — adding internal module renames to S04.20 would have ballooned its scope.
2. The internal module names are referenced from within the same service only — they have no cross-service surface. The rebrand cost is low + the deferred-hardening cost is also low.
3. Several function symbols inside these modules (`call_kraftdata`, `call_kraftdata_resilient`) are referenced from `routers/execution.py` — renaming the modules requires updating both ends of every reference, which is non-trivial.
4. This story scopes the rename to the SHARED PACKAGE per epic line 512. The deferred-hardening backlog item to rename the service-local modules is tracked separately.

If the deferred-hardening backlog item lands soon, the import statements at `routers/execution.py` line 77 (`from sirmaai_gateway.services.kraftdata_client import call_kraftdata, get_client`) and line 78 (`from sirmaai_gateway.services.kraftdata_resilient import call_kraftdata_resilient`) get updated then.

### Why we don't rename the `kraftdata-isolation-rules.yaml` Prometheus rule file

This file at `eusolicit-app/infra/observability/prometheus/rules/kraftdata-isolation-rules.yaml` defines alerts (`KraftDataDependentHighBurnRate`) and recording rules with the `slo_target="kraftdata-dependent"` label, used by `MetricsMiddleware` to isolate external-AI-provider SLO from platform SLO (per PE.05 / Story 21-5 / architecture.md line 762). The "KraftData" branding in the file name and rule names is observability-concern, not application code. Renaming requires:
1. Updating the `_KRAFTDATA_PATH_PATTERNS` regex in `services/*/middleware/metrics.py` (the path patterns that get the `slo_target="kraftdata-dependent"` label injected).
2. Updating Alertmanager routing config that fires on the rename alert.
3. Updating Grafana dashboards that reference the recording rules.
4. Cutover discipline: the new metric labels need to coexist with the old labels for a Prometheus retention window so historical dashboards keep rendering.

That's a substantial separate story. This story stays scoped to the shared Python package.

### Project memory rules honored

- **`project_sirmaai_pivot_2026_05_12.md`** ("don't recommend changes to retired surfaces"): the rename is implementation of the planned retired-surface migration per Epic 4 amendment line 512. This story is sanctioned amendment work.
- **`reference_sirmaai_api_docs.md`** ("OpenAPI at eusolicit-docs/sirmaai-reference-docs/"): AC 5's regeneration-audit consults the spec at the canonical path.
- **`project_test_execution_environment.md`** ("host venv lacks service deps; use CI for runtime"): Task 15 explicitly documents that if `make infra` is not up locally, CI is the gate — the dev pass does NOT pretend the host venv can run integration tests.
- **`project_auto_sync_quality.md`** ("Auto-sync ships unverified code"): this is a hand-authored change, not auto-sync; the dev-story agent runs the full grep audit (Task 14) before claiming completion.
- **`feedback_investigate_dont_quiz.md`** ("verify from disk first"): Task 1's pre-flight grep is the investigation; no operator questioning needed.
- **`project_sprint_status_file.md`** ("orchestrator-managed; surgical edits only"): the sprint-status row flip is a single surgical edit + audit-trail header prepend per AC 21.

### Files this story touches (definitive list — duplicated from AC 19 for in-line dev reference)

See AC 19 for the canonical list. Summary:
- Renamed: `packages/eusolicit-kraftdata/` → `packages/eusolicit-sirmaai/` (outer rename) + inner `eusolicit_kraftdata/` → `eusolicit_sirmaai/`.
- Modified inside the renamed package: `pyproject.toml`, `__init__.py`, `requests.py`, `responses.py`, `enums.py`.
- Modified consumers: 6 pyprojects + 5 Dockerfiles + workspace pyproject + docker-compose.yml + 2 CI workflows.
- Renamed test files: 4 at `tests/unit/test_eusolicit_kraftdata_*.py`.
- Modified test files: 5 scaffold-assertion files at `tests/`.
- Modified service source: 3 files (ai-gateway execution.py, sirmaai-gateway execution.py, sirmaai-gateway async_run_orchestrator.py).
- NEW: this story file.
- Modified: sprint-status.yaml (single row flip + audit-trail header prepend).

### Files this story does NOT touch (sanity list — duplicated from AC 20)

See AC 20 for the canonical list. Key deferred items:
- `infra/observability/prometheus/rules/kraftdata-isolation-rules.yaml` — separate observability story.
- `services/sirmaai-gateway/src/sirmaai_gateway/services/kraftdata_client.py` + `kraftdata_resilient.py` — separate internal-naming hardening story.
- `KRAFTDATA_*` env vars — retired alongside legacy ai-gateway code path post-cutover.
- Historical implementation-artifact files (1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md etc.) — sealed records.
- `docker-compose.prod.yml` (no kraftdata refs expected) — confirm via grep, do NOT silently edit if hits appear.
- `scripts/deploy.sh` — preserved by design per memory.
- Frontend code under `frontend/` — Python-only rename.
- Alembic migrations — no schema change.

### Test design alignment (loaded from `test_artifacts/`)

The epic-level test design at `eusolicit-docs/test-artifacts/test-design-epic-04.md` was authored 2026-04-14 (pre-SirmaAI pivot) and does NOT mention S04.30 directly. The closest analogs:

- **E04-P0-001 through E04-P0-015** — the existing P0 acceptance scenarios for the gateway's behavioural contract (happy-path sync, SSE proxy, webhook, circuit breaker, retry, rate limit, etc.). These remain canonical for the gateway's runtime behaviour and are NOT affected by this story — the rename is structural-only; the gateway's behaviour is unchanged. The S04.10 integration suite (10 scenarios) continues to pass post-rename because all import statements have been updated symmetrically.
- **§Not in Scope row at line 52** — "Kubernetes ClusterIP networking / Ingress" was the previously-out-of-scope row, recently updated by S04.29 to acknowledge the single public webhook path. Package-rename concerns are not enumerated as "out of scope" in the test design — they're orthogonal to the test design, which is behaviour-focused.
- **Test design Section §Story 1.7 references** — the original `eusolicit-kraftdata` package was test-designed in Story 1.7's ATDD checklist (`atdd-checklist-1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md`). That checklist is a sealed historical record per memory `feedback_investigate_dont_quiz.md` — it documents the original creation of the package and is NOT rewritten by this story. The four package-specific test files (renamed by AC 11) are the runtime descendants of that checklist; they continue to provide regression coverage post-rename.

The dev-pass MUST document in Completion Notes:
- AC 5 OpenAPI alignment table (the per-DTO mapping audit deliverable).
- AC 14 build-artefact decision (tracked vs untracked + cleanup command run).
- AC 15 cross-tenant invariant — covered by existing gateway test suites (S04.21 + S04.23 + S04.25), no new tests required in S04.30 scope (this is a rename, not new behaviour).
- AC 16 regression — no new failures introduced; the existing test suite stays green at the same level.
- AC 19 + AC 20 — file list adherence; any addition outside the list is justified.

### Previous story context (S04.29 immediately prior; S04.27 + S04.28 in the same amendment cluster)

S04.29 (Public Ingress for Webhook Receiver) is the most recent landed S04 story. Patterns from S04.27 / S04.28 / S04.29 carried forward here:
- **AP18-C2 atomic two-gate** (story file + sprint-status row in single dispatch): observed by this story's sprint-status flip happening atomically at story-creation time.
- **Single-commit landing**: this story's dev-pass MUST land in one commit so a rollback is a single `git revert`. The find-and-replace surface is broad (~30 files) — staging the rename across multiple commits would create intermediate states where the workspace is inconsistent (some files using the new name, others still using the old).
- **No new Prometheus counters / no new Celery tasks / no new env vars / no new migrations** — this story is entirely structural; no observability/runtime/schema surfaces are added.
- **Markdown lint discipline on docs**: not directly applicable here (no new runbook in this story), but the Completion Notes in the dev pass should be clean enough to read.
- **Pre-flight grep audit as the prerequisite gate**: encoded in Task 1.

### LLM optimization hints for the dev agent

- The single most important pre-edit is **Task 1's grep audit** + **Task 2's read-the-files-being-modified**. The rename surface is broad but mechanical; the only way to miss something is to skip the grep audit.
- The TYPE_CHECKING guard in `async_run_orchestrator.py` (Task 13) is a footgun — be careful to keep the import inside the guard.
- The `tests/unit/test_shared_packages_integration.py` file (Task 12) has the highest density of touchpoints (30+) — read it end-to-end before find-and-replacing.
- The `tests/integration/test_scaffold_integration.py` file has f-string subprocess command strings — shell-quote correctness matters. Read each touch carefully.
- The OpenAPI spec at `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json` is 493 KB — do NOT load the full file into model context. Use `jq` to enumerate `.components.schemas | keys` and selectively read individual schema components.
- The package's `pyproject.toml` only changes 1-2 lines (name + version + optional description). Don't over-edit.
- The `__init__.py` switch from `from eusolicit_kraftdata.requests import ...` (absolute) to `from .requests import ...` (relative) at AC 2 / Task 5 is a small consistency improvement that eliminates a pre-existing absolute-import anomaly inside the package — do it.
- **DO NOT** introduce camelCase field aliases, partial-update DTOs, or any other "while we're here" cleanups. The story is a rename, full stop. Scope creep on structural rename stories breaks CI in non-obvious ways.

### References

- Epic file: `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md` §S04.30 (line 512) + Amendment section (line 460+).
- Architecture amendment: `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` §ADR-018 (SirmaAI rebrand, lines 30-60).
- PRD amendment: `eusolicit-docs/planning-artifacts/prd-amendment-2026-05-12-sirmaai.md`.
- Implementation readiness report: `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-12-sirmaai.md` §Concern #3 (line 220) + §Closure index (line 414).
- SirmaAI OpenAPI spec: `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json` (consulted for AC 5 audit).
- Predecessor story (service rename — naming-rename precedent for `version` bump): `eusolicit-docs/implementation-artifacts/4-20-service-rename-and-flag-scaffold.md`.
- Predecessor stories (most recent landings — AP18-C2 atomic-two-gate pattern source): `eusolicit-docs/implementation-artifacts/4-27-tenant-degraded-mode-banner.md`, `eusolicit-docs/implementation-artifacts/4-28-tier-to-sirmaai-rate-limit-sync.md`, `eusolicit-docs/implementation-artifacts/4-29-public-ingress-for-webhook-receiver.md`.
- Historical creation record (sealed): `eusolicit-docs/implementation-artifacts/1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md` + `eusolicit-docs/test-artifacts/atdd-checklist-1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md`.
- Epic-level test design: `eusolicit-docs/test-artifacts/test-design-epic-04.md` (pre-pivot, behavioural; this story is structural-only — no test-design entry).
- Canonical files being modified (read in full before editing):
  - `eusolicit-app/packages/eusolicit-kraftdata/pyproject.toml`
  - `eusolicit-app/packages/eusolicit-kraftdata/src/eusolicit_kraftdata/{__init__,enums,requests,responses}.py`
  - `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` (TYPE_CHECKING block at lines 55-60)
  - `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` (lines 1-100)
  - `eusolicit-app/tests/unit/test_shared_packages_integration.py` (30+ touchpoints)
  - `eusolicit-app/tests/integration/test_scaffold_integration.py` (14 touchpoints, some inside f-strings)
  - `eusolicit-app/docker-compose.yml` (9 volume-mount lines)
  - `eusolicit-app/.github/workflows/ci.yml` (lines 36, 40, 44, 48, 52, 69-72)
- Memory rules observed throughout: `project_sirmaai_pivot_2026_05_12.md`, `reference_sirmaai_api_docs.md`, `project_test_execution_environment.md`, `project_auto_sync_quality.md`, `feedback_investigate_dont_quiz.md`, `project_sprint_status_file.md`.

### Project Structure Notes

- All package edits land under `eusolicit-app/packages/eusolicit-sirmaai/` (the renamed location) — no source files outside `eusolicit-app/` are touched except this story file at `eusolicit-docs/implementation-artifacts/` and the sprint-status row at `eusolicit-docs/implementation-artifacts/sprint-status.yaml`.
- The workspace `[tool.setuptools.packages.find]` array at `eusolicit-app/pyproject.toml` line 27 is the canonical workspace member list — the rename mirrors there per AC 7.
- The package's own `pyproject.toml` `[tool.setuptools.packages.find] where = ["src"]` remains unchanged — the rename of the inner directory doesn't change `find` semantics.
- No detected conflicts with unified project structure. The rename is a pure naming change at the structural boundary between packages and services.

## Dev Agent Record

### Context Reference

- [Story Context XML — TBD generated by dev-story workflow](#) <!-- placeholder; updated by dev-story dispatch -->

### Agent Model Used

claude-sonnet-4-6

### Debug Log References

None — clean pass, no unexpected errors.

### Completion Notes List

**Task 1 Pre-flight Grep Audit Results**

Pre-flight grep found 35 distinct files referencing `eusolicit-kraftdata` or `eusolicit_kraftdata`:
- 4 files inside `packages/eusolicit-kraftdata/` (the package itself — renamed)
- 1 `docker-compose.yml` (9 volume-mount lines)
- 2 CI workflow files (ci.yml, nightly.yml)
- 1 `infra/observability/prometheus/rules/kraftdata-isolation-rules.yaml` (explicitly out of scope per AC 20)
- 5 consumer pyproject.toml files
- 5 service Dockerfiles
- 1 workspace pyproject.toml
- 3 service source files (ai-gateway + sirmaai-gateway execution.py + async_run_orchestrator.py)
- 4 package-specific test files (renamed)
- 7 scaffold/assertion test files (test_scaffold_configs, test_scaffold_story_1_1, test_scaffold_integration, test_shared_packages_integration, test_ci_workflow_structure + 2 additional: test_ci_dev_dependencies, test_docker_compose_story_1_2)

Additional files outside AC 19 list: `tests/unit/test_ci_dev_dependencies.py` and `tests/unit/test_docker_compose_story_1_2.py` both had structural references to `eusolicit-kraftdata` (package path tuples + volume check assertions). Both updated. Justified addition per AC 16 exhaustive find-and-replace requirement.

Files with only sealed-historical-record references NOT updated (per `feedback_investigate_dont_quiz.md`):
- `tests/unit/test_eusolicit_models_dtos.py`, `..._enums.py`, `..._events.py`, `..._expanded.py` — docstrings referencing the story artifact file name `1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md`
- `infra/observability/prometheus/rules/kraftdata-isolation-rules.yaml` — explicitly deferred per AC 20

**AC 5 — OpenAPI Alignment Audit Table**

Spec consulted: `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json` (505 KB, 336 total schema components)

| Python Class | SirmaAI OpenAPI Component | Divergences |
|---|---|---|
| `AgentRunRequest` | `AgentRunDto` (list-view) / Not directly — closest is inferred from `AgentExecutionResponse` upstream | Intentional simplification: gateway exposes subset. Fields `message`, `session_id`, `user_id`, `tag`, `include_messages`, `tool_context` are gateway-inbound; SirmaAI API expects camelCase (`sessionId`, `userId`, `includeMessages`, `toolContext`) on the wire. Pre-existing asymmetry documented in Dev Notes — fix is separate refactor. |
| `WorkflowRunRequest` | Not directly found — `AiWorkflowRunResponse` is the response; request shape inferred | Intentional simplification: `project_id` + `tag` fields are gateway-defined. Follow-up: confirm full request body shape with SirmaAI team. |
| `TeamRunRequest` | Not directly found — `TeamExecutionResponse` is the response | Same as WorkflowRunRequest. Intentional simplification. |
| `StorageResourceRequest` | `StorageResourceCreateRequest` (name/llmConfigId/embeddingModelId) | DRIFT — HIGH: upstream `StorageResourceCreateRequest` has `name`, `llmConfigId`, `embeddingModelId` as required; our `StorageResourceRequest` has `filename`+`content_type` (file upload shape). These are different operations (create-resource vs upload-file). Gateway uses `/storage-resources/{id}/files` endpoint. File backlog: AC 5 follow-up — verify storage upload endpoint shape against spec. |
| `WebhookRegistrationRequest` | `CreateWebhookSubscriptionRequest` | DRIFT: upstream requires `name`, `endpointUrl`, `eventTypes`, `maxDeliveryAttempts`. Our model has `url`, `events`, `secret`. Field mapping: `url`→`endpointUrl`, `events`→`eventTypes`. Upstream also requires `name` + `maxDeliveryAttempts` which we don't expose. Intentional simplification — gateway wraps registration; follow-up story to align if registration is surfaced. |
| `AgentRunResponse` | `AgentJobResponseDto` (async) / `AgentExecutionResponse` (sync) | DRIFT: upstream has `run_id`, `content`, `session_id`, `user_id`, `metrics`, `messages` for sync; `jobId`, `agentId`, `status`, `userId`, `sessionId`, `result`, `error`, `startedAt`, `completedAt` for async. Our model has `execution_id`, `status`, `output`, `error`, `execution_time_ms`. Status enum divergence: upstream `AgentJobResponseDto.status` values are `PENDING/RUNNING/COMPLETED/FAILED/TIMEOUT/CANCELLED` (uppercase + extra TIMEOUT/CANCELLED) vs our `RunStatus` (lowercase, 4 values). File HIGH-priority follow-up: RunStatus enum values may need TIMEOUT + CANCELLED + case alignment. |
| `WorkflowRunResponse` | `AiWorkflowRunResponse` | DRIFT: upstream has `runId`, `workflowId`, `workflowName`, `sessionId`, `userId`, `input`, `content`, `contentType`, `result`, `messages`, `metrics`, `status`, `createdAt`. Our model has `execution_id`, `status`, `output`, `error`, `execution_time_ms`. Intentional simplification — gateway uses simplified envelope. |
| `TeamRunResponse` | `TeamRunResponseData` | DRIFT: upstream has `run_id`, `content`, `session_id`, `user_id`, `metrics`, `messages`. Our model has `execution_id`, `status`, `output`, `error`, `execution_time_ms`. Intentional simplification. |
| `StorageResourceResponse` | `StorageResourceResponse` (name matches!) | DRIFT: upstream `StorageResourceResponse` has `id`, `name`, `status`, `files`, `size`, `created`, `updated`, `crawling`, `llmConfigId`, `embeddingModelId` etc. (resource metadata). Our model has `file_id`, `filename`, `content_type`, `size_bytes`, `uploaded_at`, `resource_id` (file-upload confirmation). Different operations. Intentional simplification — see StorageResourceRequest note. |
| `WebhookEvent` | `WebhookSubscriptionResponse` / no direct inbound payload schema found | Not found — SirmaAI spec does not expose the inbound webhook payload schema in `components.schemas` (it documents subscription management, not delivery payload). Intentional hand-rolled DTO. Follow-up: validate against SirmaAI webhook delivery documentation. |
| `StreamChunk` | `ServerSentEventString` (empty schema in spec) | Intentional hand-rolled DTO. `ServerSentEventString` in spec has no properties defined. |
| `RunStatus` enum | `AgentJobResponseDto.status` enum | DRIFT — HIGH: spec enum values are `PENDING/RUNNING/COMPLETED/FAILED/TIMEOUT/CANCELLED` (uppercase); our enum is lowercase `pending/running/completed/failed` (missing TIMEOUT + CANCELLED). Since the gateway uses `RunStatus` as an inbound type hint from gateway-internal requests (NOT for deserializing SirmaAI responses), the current lowercase values match the gateway's internal naming convention. The `workflow_run_repository.py` `map_status()` function handles SirmaAI → internal status mapping. File follow-up: verify `map_status()` covers TIMEOUT and CANCELLED. |
| `ResourceType` enum | Not found in spec as an enum | Hand-rolled. Not directly in spec. Intentional. |
| `WebhookEventType` enum | `WebhookSubscriptionResponse.eventTypes` is array[string] | Hand-rolled values: `agent_completed`, `workflow_completed`, `team_completed`, `agent_failed`, `workflow_failed`, `team_failed`. No canonical enum in spec. Follow-up: verify event type string values against SirmaAI webhook delivery documentation. |

Summary: No wire-format changes made in this story (per scope constraints). HIGH-priority follow-up items: (1) RunStatus enum case + TIMEOUT/CANCELLED values, (2) StorageResourceRequest/Response shape mismatch (different operations), (3) WebhookRegistrationRequest field alignment. All deferred to separate stories per story scope constraints.

**AC 14 — Build Artefact Decision**

`git ls-files packages/eusolicit-kraftdata/build/ packages/eusolicit-kraftdata/src/eusolicit_kraftdata.egg-info/` returned EMPTY — build artefacts were UNTRACKED. After `git mv packages/eusolicit-kraftdata packages/eusolicit-sirmaai`, the untracked artifacts ended up at `packages/eusolicit-sirmaai/build/` and `packages/eusolicit-sirmaai/src/eusolicit_kraftdata.egg-info/`. Cleanup command run: `rm -rf packages/eusolicit-sirmaai/build/ packages/eusolicit-sirmaai/src/eusolicit_kraftdata.egg-info/`. Verified no kraftdata directories remain: `find packages/ -type d -name '*kraftdata*'` returned empty.

**AC 15 — Cross-Tenant Invariant**

No cross-tenant code paths touched. The package is DTO-only (Pydantic models + StrEnum enums). Cross-tenant defense remains in S04.21 + S04.23 + S04.25 unit + integration test suites. No new tests required in S04.30 scope.

**AC 16 — Regression Confirmation**

make lint: 245 errors (all pre-existing — zero stale `eusolicit_kraftdata` references in lint output; our changed files all pass `ruff check` cleanly).
make type-check: 315 errors in 94 files — all pre-existing. Only 2 errors in our touched files: yaml stubs warning in `ai-gateway/services/agent_registry.py` (pre-existing) + `execution.py:1208` arg-type in sirmaai-gateway (pre-existing from S04.28/S04.29).
make test-unit: 1813 passed, 390 failed, 330 skipped. Failures are pre-existing (helm charts missing, terraform infra missing, pagerduty test assertion, event-bus stream key tests). No new failures introduced by S04.30.
All 274 tests in our directly-changed files pass (test_eusolicit_sirmaai_*.py + test_scaffold_configs + test_shared_packages_integration + test_ci_workflow_structure + test_docker_compose_story_1_2 + test_ci_dev_dependencies).
make test-integration: Not run locally — per memory `project_test_execution_environment.md` host venv lacks service deps. CI is the canonical gate.

**AC 17 — No Backward-Compat Shim Confirmation**

Final whole-tree grep: `grep -rn 'eusolicit-kraftdata\|eusolicit_kraftdata' eusolicit-app/` shows ZERO hits in any Python/TOML/YAML/Dockerfile outside of: (a) `infra/observability/prometheus/rules/kraftdata-isolation-rules.yaml` (explicitly deferred per AC 20), (b) `packages/eusolicit-sirmaai/pyproject.toml` description string (intentional historical note), (c) sealed historical record references in docstrings pointing to `1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md` story artifact name. No alias module, no deprecation import, no symlink.

**Additional Deviations from AC 19 File List**

Two extra files updated (justified): `tests/unit/test_ci_dev_dependencies.py` + `tests/unit/test_docker_compose_story_1_2.py` — both had live structural assertions referencing `eusolicit-kraftdata` package paths. Updating them is required by AC 16 ("no new failures") and AC 17 (exhaustive find-and-replace).

Two test files updated beyond spec: `tests/integration/test_scaffold_integration.py` had a hard-coded `version == '0.1.0'` assertion for ALL packages; updated to version-agnostic check since `eusolicit-sirmaai` is now 0.2.0. Same for `tests/smoke/test_scaffold_story_1_1.py::test_shared_package_exports_version`. Also fixed `test_shared_package_importlib_metadata` version assertion (same reason).

`test_eusolicit_sirmaai_no_http_deps.py::_get_all_imports()` method patched: original logic picked up relative import `from .requests import ...` (level=1) as if it were the `requests` HTTP library (level=0). Added `node.level == 0` guard to skip relative imports. This is a correctness fix that was latent in the original file (the original `__init__.py` used an absolute import `from eusolicit_kraftdata.requests` which wouldn't have triggered the issue, but switching to relative form exposed it).

**Task 16 — CLAUDE.md Update**

Added one-paragraph entry to `eusolicit-docs/../CLAUDE.md` "Active Service Migrations" section noting the S04.30 rename.

### File List

Per AC 19 definitive list (all files touched):
- `eusolicit-app/packages/eusolicit-sirmaai/pyproject.toml` (renamed + AC 3 edits)
- `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/__init__.py` (renamed + AC 2 edits)
- `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/enums.py` (renamed + AC 4 docstring rebrand)
- `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/requests.py` (renamed + AC 4 docstring rebrand)
- `eusolicit-app/packages/eusolicit-sirmaai/src/eusolicit_sirmaai/responses.py` (renamed + AC 4 docstring rebrand)
- `eusolicit-app/services/client-api/pyproject.toml` (AC 6)
- `eusolicit-app/services/admin-api/pyproject.toml` (AC 6)
- `eusolicit-app/services/data-pipeline/pyproject.toml` (AC 6)
- `eusolicit-app/services/ai-gateway/pyproject.toml` (AC 6 legacy folder)
- `eusolicit-app/services/sirmaai-gateway/pyproject.toml` (AC 6)
- `eusolicit-app/services/notification/pyproject.toml` (AC 6)
- `eusolicit-app/pyproject.toml` (AC 7, lines 14 + 27)
- `eusolicit-app/services/client-api/Dockerfile` (AC 8)
- `eusolicit-app/services/admin-api/Dockerfile` (AC 8)
- `eusolicit-app/services/data-pipeline/Dockerfile` (AC 8)
- `eusolicit-app/services/ai-gateway/Dockerfile` (AC 8 legacy folder)
- `eusolicit-app/services/notification/Dockerfile` (AC 8)
- `eusolicit-app/docker-compose.yml` (AC 9, 9 volume-mount lines — exact count confirmed)
- `eusolicit-app/.github/workflows/ci.yml` (AC 10, lines 36/40/44/48/52 + 69-72)
- `eusolicit-app/.github/workflows/nightly.yml` (AC 10, line 109)
- `eusolicit-app/tests/unit/test_eusolicit_sirmaai_no_http_deps.py` (AC 11 rename + content)
- `eusolicit-app/tests/unit/test_eusolicit_sirmaai_responses.py` (AC 11 rename + content)
- `eusolicit-app/tests/unit/test_eusolicit_sirmaai_requests.py` (AC 11 rename + content)
- `eusolicit-app/tests/unit/test_eusolicit_sirmaai_expanded.py` (AC 11 rename + content)
- `eusolicit-app/tests/unit/test_scaffold_configs.py` (AC 12)
- `eusolicit-app/tests/smoke/test_scaffold_story_1_1.py` (AC 12)
- `eusolicit-app/tests/integration/test_scaffold_integration.py` (AC 12 + version assertion fix)
- `eusolicit-app/tests/unit/test_shared_packages_integration.py` (AC 12)
- `eusolicit-app/tests/unit/test_ci_workflow_structure.py` (AC 12)
- `eusolicit-app/tests/unit/test_ci_dev_dependencies.py` (additional — justified)
- `eusolicit-app/tests/unit/test_docker_compose_story_1_2.py` (additional — justified)
- `eusolicit-app/services/ai-gateway/src/ai_gateway/routers/execution.py` (AC 13)
- `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py` (AC 13)
- `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py` (AC 13 TYPE_CHECKING block)
- `eusolicit-docs/implementation-artifacts/4-30-package-rename-eusolicit-kraftdata-to-eusolicit-sirmaai.md` (this story file)
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` (single row flip: ready-for-dev → review)
- `/home/debian/Projects/eusolicit/CLAUDE.md` (Task 16 Active Service Migrations note)

### Test Results

**make lint**: 245 errors (all pre-existing — our changed files clean. 3 import-sort errors (I001) in `tests/unit/test_shared_packages_integration.py` were fixed with `ruff check --fix`; verified via `ruff check packages/eusolicit-sirmaai/ services/ai-gateway/src/ services/sirmaai-gateway/src/ tests/unit/test_eusolicit_sirmaai_*.py tests/unit/test_scaffold_configs.py tests/unit/test_shared_packages_integration.py tests/unit/test_ci_workflow_structure.py` → "All checks passed!")

**make type-check**: 315 errors in 94 files (all pre-existing — `python3 -m mypy packages/eusolicit-sirmaai/ + 3 service files` returns only 2 pre-existing errors in files NOT modified by this story)

**make test-unit**: 1813 passed, 390 failed, 330 skipped — all failures pre-existing (helm/terraform/pagerduty/event-bus stream-key assertions). No S04.30 regressions.

**Targeted test run (changed files)**: `274 passed, 39 skipped in 1.31s` — clean. (7 files: test_eusolicit_sirmaai_{no_http_deps,responses,requests,expanded} + test_scaffold_configs + test_shared_packages_integration + test_ci_workflow_structure)

**make test-integration**: Not run — per memory `project_test_execution_environment.md` host venv lacks service deps; CI is the canonical gate.

**make test-service SVC=sirmaai-gateway / SVC=ai-gateway**: Not run locally (requires running services); CI gate.

**Final whole-tree grep audit (Task 14)**: `grep -rn 'eusolicit-kraftdata\|eusolicit_kraftdata' eusolicit-app/` — zero hits in any non-excluded file. Remaining hits in `eusolicit-docs/implementation-artifacts/` are sealed historical records (intentional per memory `feedback_investigate_dont_quiz.md`).

**Package install verification**: `pip3 install -e packages/eusolicit-sirmaai --break-system-packages` → `Successfully installed eusolicit-sirmaai-0.2.0`. `python3 -c "import eusolicit_sirmaai; print(eusolicit_sirmaai.__version__)"` → `0.2.0`.

## Senior Developer Review

**Reviewer:** Code-review agent (BMAD adversarial review)
**Date:** 2026-05-14
**Outcome:** REVIEW: Changes Requested (low-severity cosmetic gaps in the "exhaustive find-and-replace" claim; no functional regressions)

### Scope verified

I reviewed the package rename surface end-to-end against AC 1–21 and the file list at AC 19:

- **Outer + inner `git mv` (AC 1, 2):** Confirmed — `git diff --stat` shows rename detection for all 5 package files at 70-91% similarity. `git ls-files packages/eusolicit-sirmaai/` returns the expected 5-file tree (`pyproject.toml`, `src/eusolicit_sirmaai/{__init__,enums,requests,responses}.py`). `find packages/ -type d -name '*kraftdata*'` returns empty. History preservation verified via the rename detection in the diff.
- **pyproject.toml package identity (AC 3):** `name = "eusolicit-sirmaai"`, `version = "0.2.0"`, optional `description` added — all correct.
- **`__init__.py` relative-import switch + version bump (AC 2):** Confirmed — the absolute `from eusolicit_kraftdata.requests import …` on line 7 is now `from .requests import …`, and the inconsistent enum-imports-first → requests-imports-second ordering is fixed (now alphabetical: enums, requests, responses). `__version__ = "0.2.0"`. `__all__` preserved.
- **Docstring rebrand inside the package (AC 4):** Verified — `grep -n KraftData packages/eusolicit-sirmaai/src/` returns zero hits. Module docstrings and per-class docstrings all swap KraftData → SirmaAI. `WebhookEvent` docstring path correctly updated to `POST /webhooks/sirmaai` (S04.25 correction).
- **OpenAPI alignment audit (AC 5):** The per-DTO mapping table in Completion Notes covers all 11 model classes + 3 enums with HIGH-priority drift flagged for `RunStatus`/`StorageResourceRequest`/`WebhookRegistrationRequest`. Deliverable accepted.
- **Consumer pyproject deps (AC 6) + workspace pyproject (AC 7) + Dockerfiles (AC 8) + docker-compose (AC 9) + CI workflows (AC 10):** All cleanly updated; `grep` audit confirms zero `eusolicit-kraftdata`/`eusolicit_kraftdata` hits in any non-excluded structural file.
- **Test renames (AC 11) + scaffold-assertion updates (AC 12):** All four `git mv` renames present with content updates. The two additional files (`test_ci_dev_dependencies.py`, `test_docker_compose_story_1_2.py`) are justified per AC 16's exhaustive-find-and-replace clause.
- **Service-level import updates (AC 13):** Confirmed — `services/ai-gateway/src/ai_gateway/routers/execution.py`, `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py`, and the `TYPE_CHECKING:` guard at `services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py:56` all updated. The TYPE_CHECKING import correctly stays inside the guard (cycle preserved).
- **Build-artefact cleanup (AC 14):** `git ls-files` of the renamed package returns only the 5 source files. The untracked `packages/eusolicit-sirmaai/src/eusolicit_sirmaai.egg-info/` directory in the working tree appears post-rename from a local `pip install -e` verification step; it is not staged and will not land in the commit.
- **No-shim discipline (AC 17):** Confirmed — no alias module, no deprecation re-export.
- **DoD gates (AC 18):** Lint/type-check clean on touched files; 274 targeted tests pass; pre-existing failures documented and not regressions.

### Findings

#### LOW (cosmetic — exhaustive-find-and-replace gaps)

1. **`tests/unit/test_eusolicit_sirmaai_expanded.py:395` — class name not rebranded.**
   `class TestKraftdataPackageExports:` was not renamed to `TestSirmaaiPackageExports`, even though every method body, docstring, and assertion inside the class now uses `eusolicit_sirmaai` / "SirmaAI" prose. The class identifier is the last KraftData-branded symbol the rename surface kept on disk. AC 4's "no field-name changes, no class-name changes" carve-out applies to the *package's own* DTO/enum classes, not to the test-file class organising the renamed test methods.
   **Fix:** rename the class to `TestSirmaaiPackageExports`.

2. **`tests/unit/test_eusolicit_sirmaai_requests.py:226, 229` — stale webhook URL fixture.**
   The `TestWebhookRegistrationRequest` test constructs `url="https://eusolicit.example.com/webhooks/kraftdata"` and asserts the round-trip on that string. After S04.25 the actual receiver path is `/webhooks/sirmaai`, and AC 4's `responses.py:WebhookEvent` docstring was updated to match. Leaving the test fixture URL at `/webhooks/kraftdata` is a stale string in a renamed-test surface and conflicts with the dev's own Task-6 "find-and-replace must be exhaustive" claim.
   **Fix:** update both occurrences to `/webhooks/sirmaai`.

3. **`tests/unit/test_eusolicit_sirmaai_expanded.py:18` — broken historical-record reference.**
   The docstring was edited from `1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md` to `1-7-eusolicit-models-eusolicit-sirmaai-shared-packages.md`, but no such file exists in `eusolicit-docs/implementation-artifacts/`. The actual historical filename is the kraftdata-suffixed one (sealed historical record per memory `feedback_investigate_dont_quiz.md`). The dev's Completion Notes acknowledge this policy for `test_eusolicit_models_*.py` files but apply the opposite policy to this renamed sibling, producing a dangling reference.
   The companion file `test_eusolicit_sirmaai_no_http_deps.py:12` correctly retains the historical kraftdata filename — these two renamed siblings now disagree.
   **Fix:** revert the path on `_expanded.py:18` back to `1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md` for consistency with `_no_http_deps.py` and with the actual on-disk filename.

#### INFORMATIONAL

4. **Working-tree scope discipline at commit time.**
   The current `git status` working tree includes substantial non-S04.30 changes (frontend `tsbuildinfo`, `messages/{bg,en}.json`, `apps/client/.../layout.tsx`; `infra/nginx/` + `infra/host/templates/nginx-sites/`; sirmaai-gateway internals — `routers/admin.py`, `services/circuit_breaker.py`, `services/exceptions.py`, `services/rate_limit_sync_consumer.py`, `tasks/reconcile_runs.py`, plus the new migration `006_rate_limit_sync_audit.py`, plus the expanded integration suites for those features; `tests/integration/test_db_schema_isolation.py`; `services/client-api/src/client_api/{main.py,models/sirmaai_project.py,services/ai_gateway_client.py}`). Some of those may belong to S04.28/S04.29 follow-up work and not to S04.30.
   The Completion Notes claim "single-commit landing" per AP18-C2 atomic discipline, but unless the operator stages strictly the S04.30 file list (AC 19) the rename will land mixed with unrelated work, defeating the single-revert rollback property.
   **Action:** before commit, stage only the AC 19 file list (plus the two AC-16-justified additions and the CLAUDE.md crumb). The remaining working-tree changes should land on their own commits under their own story keys.

5. **`packages/eusolicit-sirmaai/src/eusolicit_sirmaai.egg-info/` untracked.**
   Present in the renamed package directory from the local `pip install -e` verification. Untracked, will not enter the commit. No action required, just noting the workspace cleanliness.

### Architecture & invariants

- **Schema isolation:** unchanged; this is a Python package rename, no DB or schema touches.
- **Cross-tenant defense (AC 15):** correctly untouched — package is DTO-only.
- **Service-internal `kraftdata_client.py` / `kraftdata_resilient.py` / `_resolve_kraftdata_id()` helpers:** correctly left untouched per AC 13 + AC 20 (separate deferred-hardening story).
- **`kraftdata-isolation-rules.yaml`:** correctly untouched per AC 20.
- **Legacy `services/ai-gateway/` folder:** symmetrically updated alongside `sirmaai-gateway/` (pyproject + Dockerfile + execution.py imports), preserving the S04.20 flag-off cutover path. Correct.
- **TYPE_CHECKING guard:** preserved at `async_run_orchestrator.py:55-60`. Correct.
- **`__init__.py` absolute → relative import switch:** correct improvement; eliminates a latent inconsistency.

### Test coverage

- 274 directly-changed tests pass per Completion Notes.
- Lint clean on touched files; type-check clean on touched files (2 pre-existing errors documented as non-regressions).
- `make test-integration` deferred to CI per project memory `project_test_execution_environment.md`. Acceptable; CI is canonical.

### Recommendation

**REVIEW: Changes Requested** — apply the three LOW findings (class rename, webhook URL fixture, historical-record reference) and stage strictly per AC 19 before commit. The rename surface is otherwise structurally sound; once the three cosmetic touchups land, the story is ready to flip to `done`.

### Post-Review Fix Record (2026-05-14)

All three LOW findings applied:

1. **`tests/unit/test_eusolicit_sirmaai_expanded.py:395`** — `TestKraftdataPackageExports` → `TestSirmaaiPackageExports`.
2. **`tests/unit/test_eusolicit_sirmaai_requests.py:226, 229`** — webhook URL fixture `/webhooks/kraftdata` → `/webhooks/sirmaai`.
3. **`tests/unit/test_eusolicit_sirmaai_expanded.py:18`** — reverted dangling reference `1-7-eusolicit-models-eusolicit-sirmaai-shared-packages.md` back to the correct sealed historical filename `1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md`, consistent with `test_eusolicit_sirmaai_no_http_deps.py:12`.

**Post-fix test results**: `274 passed, 39 skipped in 1.35s` — unchanged, no regressions.
**Lint**: `ruff check` on both modified files → `All checks passed!`

### Audit Pass-2 (2026-05-14)

**Reviewer:** Code-review agent (BMAD adversarial re-review)
**Outcome:** **REVIEW: Approve**

Fresh adversarial pass against the on-disk state after the Post-Review Fix Record was applied. Verified:

- **Pass-1 LOW findings all closed on disk:**
  - `tests/unit/test_eusolicit_sirmaai_expanded.py:395` — `class TestSirmaaiPackageExports:` confirmed.
  - `tests/unit/test_eusolicit_sirmaai_requests.py:226, 229` — `url="https://eusolicit.example.com/webhooks/sirmaai"` confirmed (both occurrences).
  - `tests/unit/test_eusolicit_sirmaai_expanded.py:18` and `_no_http_deps.py:12` — both consistently reference the sealed historical filename `1-7-eusolicit-models-eusolicit-kraftdata-shared-packages.md`.
- **Package surface (AC 1–4):** `git mv` renames preserved; `packages/eusolicit-sirmaai/src/eusolicit_sirmaai/` tree contains only the 4 source files + `__init__.py`; `name = "eusolicit-sirmaai"`, `version = "0.2.0"`; `__init__.py` uses relative imports throughout, alphabetised (enums → requests → responses); `__all__` lists all 14 exports. `grep KraftData packages/eusolicit-sirmaai/src/` returns zero hits.
- **Consumer surfaces (AC 6–10):** `grep eusolicit-kraftdata` across `docker-compose.yml`, `pyproject.toml`, `services/*/pyproject.toml`, `services/*/Dockerfile`, `.github/workflows/{ci,nightly}.yml` returns zero hits. docker-compose volume mounts count = 9 (confirmed). CI matrix entry `eusolicit-sirmaai` correctly paired with `test-filter: eusolicit_sirmaai` and the 4 renamed test files; per-service `deps:` lines on L36/40/44/48/52 all rewrite the package install path correctly.
- **Service-level imports (AC 13):** `from eusolicit_sirmaai.requests/responses import …` in both `services/ai-gateway/src/ai_gateway/routers/execution.py` and `services/sirmaai-gateway/src/sirmaai_gateway/routers/execution.py`. The `TYPE_CHECKING:` guard at `services/sirmaai-gateway/src/sirmaai_gateway/services/async_run_orchestrator.py:54-60` is preserved — the renamed import stays inside the guard (cycle-breaker invariant intact).
- **Deferred surfaces correctly untouched (AC 20):** `services/ai-gateway/src/ai_gateway/routers/execution.py` retains its service-internal `KraftData*` exception class imports + `_resolve_kraftdata_id` / `_handle_kraftdata_error` helpers as expected; `infra/observability/prometheus/rules/kraftdata-isolation-rules.yaml` unmodified; `services/sirmaai-gateway/src/sirmaai_gateway/services/kraftdata_client.py` / `kraftdata_resilient.py` unmodified.
- **Runtime smoke (AC 18 subset):** `python3 -c "import eusolicit_sirmaai; print(eusolicit_sirmaai.__version__)"` → `0.2.0`. `python3 -m py_compile` clean on all four source files. `ruff check packages/eusolicit-sirmaai/ tests/unit/test_eusolicit_sirmaai_*.py` → "All checks passed!". `pytest` on the 7 directly-changed test files → `274 passed, 39 skipped in 1.32s`.
- **Working-tree contamination (Pass-1 INFORMATIONAL #4):** the working tree continues to carry non-S04.30 changes (sirmaai-gateway internals from S04.28/S04.29 follow-up, frontend banner work, infra/nginx). This is unchanged from Pass-1's INFORMATIONAL note and is an operator-stage discipline concern, not a code-quality finding against S04.30. The dev's Completion Notes claim a single-commit landing per AP18-C2; that contract is preserved only if the commit is staged strictly against the AC 19 file list (plus the two AC-16-justified additions plus the CLAUDE.md crumb).

No new findings. The rename surface is structurally sound and the post-fix state is stable. Story remains at `Status: done`.
**Remaining `kraftdata` in modified files**: only the sealed historical reference at `expanded.py:18` (correct per `feedback_investigate_dont_quiz.md`).
