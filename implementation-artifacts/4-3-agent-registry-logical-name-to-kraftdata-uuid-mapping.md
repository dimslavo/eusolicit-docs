# Story 4.3: Agent Registry — Logical Name to KraftData UUID Mapping

Status: review

## Change Log

- 2026-04-14: Story created from epic-04-ai-gateway-service.md S04.03 with test design context from test-design-epic-04.md. Agent: Claude Sonnet 4.6
- 2026-04-14: Story implemented — all 7 tasks complete, 26/26 tests pass, ruff clean. Agent: Claude Sonnet 4.6

## Story

As a **backend developer on the EU Solicit platform**,
I want **a configuration-driven `AgentRegistry` that maps logical agent names to KraftData UUIDs, validates the mapping on startup, and supports hot-reload without service restart**,
so that **all subsequent gateway stories (S04.04–S04.09) can resolve agents by human-readable names, KraftData UUID swaps become a config change instead of a code change, and unknown agents fail fast with a 404 rather than silently misrouting to the wrong KraftData resource**.

## Acceptance Criteria

1. `AgentRegistry` loads `config/agents.yaml` on startup; all 29 entries (27 agents, 1 team, 1 workflow) load without error
2. `registry.resolve("executive-summary")` returns an `AgentEntry` with the correct `kraftdata_id` (UUID string) and `type="agent"`
3. `registry.resolve("nonexistent-agent")` raises `AgentNotFoundError`; the FastAPI exception handler maps this to HTTP 404 with body `{"error": "agent_not_found", "logical_name": "<name>"}`
4. Duplicate logical names in `agents.yaml` cause startup to fail fast with a `ValueError` naming the duplicate key; the service does not start
5. Pydantic validates every entry on load: `kraftdata_id` must match UUID v4 format, `type` must be `agent | team | workflow`; invalid entries raise `ValueError` with the offending key named
6. `registry.list_agents()` returns all loaded `AgentEntry` instances sorted by name (for admin introspection)
7. `POST /admin/registry/reload` reloads `agents.yaml` from disk; on success returns 200 with `{"loaded": <count>}`; on validation failure returns 400 with `{"error": "<details>"}` and the existing registry remains intact
8. Hot-reload is concurrency-safe: concurrent `resolve()` calls during a reload complete without exception or partial data; an `asyncio.Lock` guards the `_entries` dict swap
9. `AgentNotFoundError` is defined in `services/exceptions.py` and re-exported from `services/__init__.py` alongside the existing KraftData exception hierarchy
10. Registry singleton is initialized in `main.py` lifespan startup (after `init_client`) via `await init_registry(Path(settings.agents_yaml_path))`; service fails fast if YAML is missing or invalid
11. Admin router registered at `app.include_router(admin_router.router, prefix="/admin")` — reload and list endpoints live under `/admin`
12. `GET /admin/registry` returns the full list of loaded entries with `name`, `kraftdata_id`, `type`, `description`, and optional `timeout_override` and `max_concurrent`

## Tasks / Subtasks

- [x] Task 1: Extend `src/ai_gateway/services/exceptions.py` with `AgentNotFoundError` (AC: 3, 9)
  - [x] 1.1 Define `AgentNotFoundError(Exception)` with `logical_name: str` attribute; message: `f"Agent '{logical_name}' not found in registry"`
  - [x] 1.2 Add `AgentNotFoundError` to `__all__` in `services/__init__.py` and add the import re-export alongside the four KraftData exceptions

- [x] Task 2: Create `src/ai_gateway/services/agent_registry.py` — Pydantic models and `AgentRegistry` class (AC: 1–8, 10)
  - [x] 2.1 Define `AgentEntry` Pydantic model with fields: `name: str = ""` (set by loader from YAML key), `kraftdata_id: str` (UUID v4 validated via `@field_validator`), `type: Literal["agent", "team", "workflow"]`, `description: str = ""`, `timeout_override: int | None = None`, `max_concurrent: int | None = None`
  - [x] 2.2 Implement `_NoDuplicatesLoader` — a `yaml.SafeLoader` subclass that overrides the default mapping constructor to raise `ValueError` on duplicate keys (standard `yaml.safe_load` silently keeps the last value; this custom loader catches duplicates at the YAML parse level)
  - [x] 2.3 Implement static method `AgentRegistry._parse_yaml(yaml_path: Path) -> dict[str, AgentEntry]`:
    - Read and `yaml.load(text, Loader=_NoDuplicatesLoader)` — raises `ValueError` on duplicate keys
    - Validate top-level `"agents"` key exists
    - For each key/data pair: call `AgentEntry.model_validate(data)` then set `name=key` via `model_copy`; collect into dict
    - Wrap Pydantic `ValidationError` as `ValueError` with the offending agent name
  - [x] 2.4 Implement `AgentRegistry` class:
    - `__init__(self, yaml_path: Path)` — stores `_yaml_path`, initializes `_entries: dict[str, AgentEntry] = {}`, `_lock: asyncio.Lock = asyncio.Lock()`
    - `async def load(self) -> None` — calls `_parse_yaml`, assigns to `_entries`; raises on error
    - `async def reload(self) -> int` — acquires `_lock`, calls `_parse_yaml` (validation errors raise before any swap), atomically replaces `_entries`, releases lock; returns new count
    - `def resolve(self, logical_name: str) -> AgentEntry` — returns entry or raises `AgentNotFoundError(logical_name)` (no lock needed — dict reference reads are atomic in CPython)
    - `def list_agents(self) -> list[AgentEntry]` — returns `sorted(self._entries.values(), key=lambda e: e.name)`
    - `@property def count(self) -> int` — returns `len(self._entries)`
  - [x] 2.5 Implement module-level singleton: `_registry: AgentRegistry | None = None`
  - [x] 2.6 Implement `async def init_registry(yaml_path: Path) -> None` — creates `AgentRegistry`, calls `.load()`, assigns to `_registry`; propagates exceptions so lifespan fails fast
  - [x] 2.7 Implement `def get_registry() -> AgentRegistry` — returns `_registry`; raises `RuntimeError("AgentRegistry not initialised")` if called before `init_registry()`

- [x] Task 3: Create `config/agents.yaml` — 29-entry registry (AC: 1)
  - [x] 3.1 Write full YAML with 27 agents, 1 team (`proposal-generation-team`), 1 workflow (`full-proposal-workflow`); include placeholder UUID v4 strings for `kraftdata_id`; set `timeout_override` on slow agents (e.g., `proposal-drafter: 300`, `full-proposal-workflow: 900`); set `max_concurrent: 3` on 2 high-traffic agents (`espd-auto-fill`, `consortium-finder`)
  - [x] 3.2 Remove the `config/.gitkeep` stub (created by S04.01) and commit `agents.yaml` in its place
  - [x] 3.3 Verify YAML parses cleanly with the `_NoDuplicatesLoader` and all 29 entries pass Pydantic validation

- [x] Task 4: Add `agents_yaml_path` setting to `config.py` (AC: 10)
  - [x] 4.1 Add `agents_yaml_path: str = "config/agents.yaml"` to `AIGatewaySettings` class body (relative to the uvicorn working directory — `services/ai-gateway/`)

- [x] Task 5: Wire registry into `main.py` lifespan and register admin router (AC: 10, 11)
  - [x] 5.1 Import `init_registry`, `get_registry` from `ai_gateway.services.agent_registry`
  - [x] 5.2 In lifespan startup (after `await init_client(settings)`): call `await init_registry(Path(settings.agents_yaml_path))`; add `log.info("registry.loaded", count=get_registry().count)` for startup observability (E04-R-009 mitigation — every UUID loaded is logged on startup)
  - [x] 5.3 Import admin router and register: `app.include_router(admin_router.router, prefix="/admin")`
  - [x] 5.4 Add FastAPI exception handler for `AgentNotFoundError` → HTTP 404 with `{"error": "agent_not_found", "logical_name": exc.logical_name}`

- [x] Task 6: Create `src/ai_gateway/routers/admin.py` — admin endpoints for registry introspection and reload (AC: 7, 11, 12)
  - [x] 6.1 Create `router = APIRouter(tags=["Admin"])`
  - [x] 6.2 Implement `GET /registry` with `response_model=list[AgentEntry]` — calls `registry.list_agents()`; HTTP 200
  - [x] 6.3 Implement `POST /registry/reload` — calls `await registry.reload()`; on success returns `{"loaded": count}`; on any `Exception` returns `JSONResponse(status_code=400, content={"error": str(exc)})`
  - [x] 6.4 Both endpoints take `registry: AgentRegistry = Depends(_get_registry)` where `_get_registry` wraps `get_registry()`

- [x] Task 7: Write unit tests — `tests/unit/test_agent_registry.py` (AC: 1–8; test IDs E04-P0-004, E04-P1-017, E04-P2-001, E04-P2-002, E04-P2-003)
  - [x] 7.1 `test_load_valid_yaml` (E04-P0-004): use `tmp_path` to write a 5-entry fixture YAML (3 agents, 1 team, 1 workflow); assert `registry.count == 5`; assert no exception raised
  - [x] 7.2 `test_resolve_known_agent` (E04-P0-004): assert `registry.resolve("executive-summary").kraftdata_id == EXPECTED_UUID`; assert `.type == "agent"`; assert `.name == "executive-summary"`
  - [x] 7.3 `test_resolve_unknown_raises` (E04-P0-004): assert `registry.resolve("nonexistent")` raises `AgentNotFoundError`; assert `exc.logical_name == "nonexistent"`
  - [x] 7.4 `test_duplicate_key_raises_on_load` (E04-P1-017): write fixture YAML with duplicate key `executive-summary` (two entries with the same name); assert `await registry.load()` raises `ValueError`; assert error message includes the string `"executive-summary"`
  - [x] 7.5 `test_timeout_override_fallback`: assert entry without `timeout_override` has `.timeout_override is None`; assert entry with `timeout_override: 180` has `.timeout_override == 180`
  - [x] 7.6 `test_list_agents_returns_all`: assert `len(registry.list_agents()) == 5`; assert all entries have `.name` populated; assert list is sorted alphabetically by `.name`
  - [x] 7.7 `test_reload_updates_registry` (E04-P2-001): load registry with YAML-A (has `agent-a`); write YAML-B (has `agent-b`, not `agent-a`) to same path; call `await registry.reload()`; assert `registry.resolve("agent-b")` succeeds; assert `registry.resolve("agent-a")` raises `AgentNotFoundError`; assert `registry.count == 1`
  - [x] 7.8 `test_reload_invalid_yaml_keeps_old_registry` (E04-P2-002): load valid registry (1 entry); overwrite YAML with invalid content (bad UUID); call `await registry.reload()`; assert `reload()` raises `ValueError`; assert `registry.resolve("executive-summary")` still succeeds (old entries intact)
  - [x] 7.9 `test_concurrent_resolve_during_reload` (E04-P2-003): create a registry with 5 entries; launch 20 concurrent `resolve("executive-summary")` tasks with `asyncio.gather`; fire `reload()` concurrently via a separate task mid-gather; assert all tasks complete without exception; assert no `KeyError` or `AttributeError` propagated
  - [x] 7.10 `test_invalid_uuid_format_raises` (AC: 5): write YAML with `kraftdata_id: "not-a-valid-uuid"`; assert `await registry.load()` raises `ValueError` mentioning the invalid UUID
  - [x] 7.11 `test_invalid_type_raises` (AC: 5): write YAML with `type: "invalid-type"`; assert `await registry.load()` raises `ValueError` mentioning the offending entry

### Senior Developer Review

**Date:** 2026-04-14 | **Reviewer:** Claude Opus 4.6 (bmad-code-review) | **Verdict: APPROVE**

All 12 acceptance criteria verified. All 14 unit tests pass. Full test suite (30/30 + 1 skip) green. Ruff clean. No regressions.

**Findings:**

- [ ] [Review][Patch] Redundant exception tuple `(ValidationError, Exception)` — `Exception` subsumes `ValidationError` [agent_registry.py:218]
- [ ] [Review][Patch] Dead assertion in `test_timeout_override_fallback` — line 141 `assert ... == 180 or ... == 120` is always true if line 143 passes [test_agent_registry.py:141]
- [x] [Review][Defer] `reload()` reads `len(self._entries)` outside the lock scope (line 153-154) — deferred, pre-existing design pattern; single-lock-holder guarantee makes this safe

**AC Coverage Matrix:** AC 1–12 all ✅ | Test IDs E04-P0-004, E04-P1-017, E04-P2-001, E04-P2-002, E04-P2-003 all ✅

## Dev Notes

### CRITICAL: Module Path Convention (same as S04.01/S04.02)

The epic spec uses `app/services/agent_registry.py` and `config/agents.yaml` — the **actual** monorepo paths are:

| Epic spec path | Actual monorepo path |
|---|---|
| `app/services/agent_registry.py` | `services/ai-gateway/src/ai_gateway/services/agent_registry.py` |
| `config/agents.yaml` | `services/ai-gateway/config/agents.yaml` |
| Admin router | `services/ai-gateway/src/ai_gateway/routers/admin.py` |

Do NOT create a top-level `app/` directory. The `config/.gitkeep` stub was created by S04.01 — delete it and place `agents.yaml` in the same `config/` directory.

### What Already Exists After S04.01 and S04.02

| File | State | Action for S04.03 |
|---|---|---|
| `src/ai_gateway/services/exceptions.py` | Has `KraftDataError`, `KraftDataTimeoutError`, `KraftDataConnectionError`, `KraftDataAPIError` | **EXTEND** — add `AgentNotFoundError` |
| `src/ai_gateway/services/__init__.py` | Re-exports 4 KraftData exceptions | **EXTEND** — add `AgentNotFoundError` re-export |
| `src/ai_gateway/config.py` | Complete with KraftData + concurrency settings | **EXTEND** — add `agents_yaml_path` field |
| `src/ai_gateway/main.py` | Lifespan calls `init_client`/`close_client`; registers `health_router` | **EXTEND** — add `init_registry` in lifespan; register admin router; add exception handler for `AgentNotFoundError` |
| `src/ai_gateway/routers/__init__.py` | Empty `__init__.py` | **DO NOT TOUCH** |
| `src/ai_gateway/routers/health.py` | `/health` and `/ready` endpoints | **DO NOT TOUCH** |
| `config/.gitkeep` | Placeholder stub from S04.01 | **DELETE** — replace with `agents.yaml` |
| `pyproject.toml` | Has `pyyaml>=6.0` already in dependencies | **DO NOT TOUCH** |
| `tests/conftest.py` | DB engine + session + `mock_kraftdata_base_url` fixtures | **DO NOT TOUCH** |

### `AgentNotFoundError` Design

```python
# src/ai_gateway/services/exceptions.py — addition (append to existing file)
class AgentNotFoundError(Exception):
    """Raised when a logical agent name is not found in the registry."""

    def __init__(self, logical_name: str) -> None:
        self.logical_name = logical_name
        super().__init__(f"Agent '{logical_name}' not found in registry")
```

Add the FastAPI exception handler in `main.py`:

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from ai_gateway.services.exceptions import AgentNotFoundError

@app.exception_handler(AgentNotFoundError)
async def agent_not_found_handler(request: Request, exc: AgentNotFoundError) -> JSONResponse:
    return JSONResponse(
        status_code=404,
        content={"error": "agent_not_found", "logical_name": exc.logical_name},
    )
```

### Duplicate Key Detection in YAML

Standard `yaml.safe_load` silently keeps the **last** value when duplicate keys exist — it does not raise. A custom loader is required to catch this. Implement a no-duplicates YAML loader:

```python
# Inside agent_registry.py

class _NoDuplicatesLoader(yaml.SafeLoader):
    """YAML loader that raises ValueError on duplicate mapping keys."""


def _construct_no_dup_mapping(
    loader: _NoDuplicatesLoader, node: yaml.MappingNode
) -> dict:
    loader.flatten_mapping(node)
    seen: dict[str, None] = {}
    for key_node, _ in node.value:
        key = loader.construct_object(key_node)
        if key in seen:
            raise ValueError(f"Duplicate key in agents.yaml: '{key}'")
        seen[key] = None
    return loader.construct_pairs(node)  # let pyyaml finish normal construction


_NoDuplicatesLoader.add_constructor(
    yaml.resolver.BaseResolver.DEFAULT_MAPPING_TAG,
    _construct_no_dup_mapping,
)
```

**Note:** The constructor above detects the duplicate during `yaml.load()` and raises `ValueError` before any data is returned to `_parse_yaml`. The caller sees a `ValueError` with the duplicate key name.

### `AgentEntry` Pydantic Model

```python
import re
from typing import Literal
from pydantic import BaseModel, field_validator

_UUID4_RE = re.compile(
    r"^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$",
    re.IGNORECASE,
)


class AgentEntry(BaseModel):
    """Single registry entry: logical name → KraftData UUID."""

    name: str = ""  # populated from YAML key by loader; not present in YAML body
    kraftdata_id: str
    type: Literal["agent", "team", "workflow"]
    description: str = ""
    timeout_override: int | None = None   # seconds; None = use KraftData client default (120s read)
    max_concurrent: int | None = None     # per-agent concurrency cap for S04.09; None = no limit

    @field_validator("kraftdata_id")
    @classmethod
    def validate_uuid4(cls, v: str) -> str:
        if not _UUID4_RE.match(v):
            raise ValueError(f"kraftdata_id '{v}' is not a valid UUID v4")
        return v
```

### `AgentRegistry` Class Skeleton

```python
import asyncio
from pathlib import Path
import structlog
import yaml

log = structlog.get_logger(__name__)
_registry: AgentRegistry | None = None


class AgentRegistry:
    def __init__(self, yaml_path: Path) -> None:
        self._yaml_path = yaml_path
        self._entries: dict[str, AgentEntry] = {}
        self._lock: asyncio.Lock = asyncio.Lock()

    async def load(self) -> None:
        self._entries = self._parse_yaml(self._yaml_path)
        log.info("registry.loaded", count=len(self._entries), path=str(self._yaml_path))

    async def reload(self) -> int:
        async with self._lock:
            # _parse_yaml raises BEFORE any swap on error — old registry stays intact
            new_entries = self._parse_yaml(self._yaml_path)
            self._entries = new_entries  # atomic reference assignment
        count = len(self._entries)
        log.info("registry.reloaded", count=count)
        return count

    def resolve(self, logical_name: str) -> AgentEntry:
        entry = self._entries.get(logical_name)
        if entry is None:
            raise AgentNotFoundError(logical_name)
        return entry

    def list_agents(self) -> list[AgentEntry]:
        return sorted(self._entries.values(), key=lambda e: e.name)

    @property
    def count(self) -> int:
        return len(self._entries)

    @staticmethod
    def _parse_yaml(yaml_path: Path) -> dict[str, AgentEntry]:
        if not yaml_path.exists():
            raise ValueError(f"agents.yaml not found at '{yaml_path}'")
        text = yaml_path.read_text(encoding="utf-8")
        try:
            raw = yaml.load(text, Loader=_NoDuplicatesLoader)  # raises on duplicate keys
        except ValueError:
            raise  # duplicate key error from our custom loader
        except yaml.YAMLError as exc:
            raise ValueError(f"YAML parse error: {exc}") from exc
        if not isinstance(raw, dict) or "agents" not in raw:
            raise ValueError("agents.yaml must have a top-level 'agents' key")
        agents_raw: dict = raw["agents"]
        entries: dict[str, AgentEntry] = {}
        for key, data in agents_raw.items():
            try:
                entry = AgentEntry.model_validate(data)
                entry = entry.model_copy(update={"name": key})
                entries[key] = entry
            except Exception as exc:
                raise ValueError(f"Invalid entry for agent '{key}': {exc}") from exc
        return entries


async def init_registry(yaml_path: Path) -> None:
    global _registry
    _registry = AgentRegistry(yaml_path)
    await _registry.load()


def get_registry() -> AgentRegistry:
    if _registry is None:
        raise RuntimeError(
            "AgentRegistry not initialised. Call init_registry() during app startup."
        )
    return _registry
```

### Hot-Reload Concurrency Safety (E04-R-006)

The `asyncio.Lock` in `reload()` prevents two concurrent reload calls from racing against each other. The `resolve()` method does **not** acquire the lock — it reads `self._entries` which is a dict reference. In CPython, assigning a new dict to `self._entries` is a single bytecode instruction (`STORE_ATTR`) that is atomic under the GIL. The lock prevents interleaved partial reloads, not partial reads.

This is safe for a single-asyncio-event-loop process. See E04-R-005 (in test-design-epic-04.md) for the known limitation when multi-replica deployment is introduced.

### `config.py` Addition

```python
# Add to AIGatewaySettings class body
agents_yaml_path: str = "config/agents.yaml"
```

This path is relative to the working directory where uvicorn is launched (`services/ai-gateway/`). In Kubernetes production, mount the YAML as a ConfigMap at this path and use `AGENTS_YAML_PATH` env override.

### Updated `main.py` Lifespan Pattern

```python
# src/ai_gateway/main.py — updated lifespan (S04.03 additions marked)
from pathlib import Path
from fastapi import Request
from fastapi.responses import JSONResponse
from ai_gateway.services.agent_registry import init_registry, get_registry
from ai_gateway.services.exceptions import AgentNotFoundError
from ai_gateway.routers import admin as admin_router

import structlog
log = structlog.get_logger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    settings = get_settings()
    setup_logging("ai-gateway", environment=settings.environment, log_level=settings.log_level)
    await init_client(settings)                                    # S04.02
    await init_registry(Path(settings.agents_yaml_path))          # S04.03 — fail fast
    log.info("registry.loaded", count=get_registry().count)       # S04.03 — E04-R-009 observability
    yield
    await close_client()                                           # S04.02

# After app = FastAPI(...) and register_exception_handlers(app):
app.include_router(health_router.router)
app.include_router(admin_router.router, prefix="/admin")           # S04.03

@app.exception_handler(AgentNotFoundError)                        # S04.03
async def agent_not_found_handler(request: Request, exc: AgentNotFoundError) -> JSONResponse:
    return JSONResponse(
        status_code=404,
        content={"error": "agent_not_found", "logical_name": exc.logical_name},
    )
```

### `config/agents.yaml` — Full 29-Entry Registry

All `kraftdata_id` values below are placeholder UUID v4 strings. Replace with real KraftData UUIDs before going live. See E04-R-009 for the UUID rotation runbook.

```yaml
# EU Solicit — AI Gateway Agent Registry
# 27 agents + 1 team + 1 workflow = 29 entries total
# Replace all kraftdata_id values with real KraftData UUIDs before production deployment.
agents:

  # ── Core proposal agents ────────────────────────────────────────────────────
  executive-summary:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000001"
    type: agent
    description: "Generates executive summary from tender documents"
    timeout_override: 120

  espd-auto-fill:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000002"
    type: agent
    description: "Auto-fills ESPD profile fields from company data"
    timeout_override: 180
    max_concurrent: 3

  grant-eligibility:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000003"
    type: agent
    description: "Assesses company eligibility for a specific grant call"
    timeout_override: 90

  budget-builder:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000004"
    type: agent
    description: "Constructs detailed budget breakdown from project parameters"
    timeout_override: 150

  consortium-finder:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000005"
    type: agent
    description: "Identifies potential consortium partners matching project requirements"
    max_concurrent: 3

  logframe-generator:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000006"
    type: agent
    description: "Generates logical framework matrix from project objectives"
    timeout_override: 120

  reporting-template:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000007"
    type: agent
    description: "Produces reporting templates aligned to funder requirements"

  regulation-tracker:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000008"
    type: agent
    description: "Tracks relevant EU regulatory updates affecting active grants"

  # ── Tender analysis agents ──────────────────────────────────────────────────
  tender-summarizer:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000009"
    type: agent
    description: "Summarizes tender call documents into actionable briefings"

  requirements-extractor:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000010"
    type: agent
    description: "Extracts structured eligibility and technical requirements from tender PDFs"

  eligibility-checker:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000011"
    type: agent
    description: "Cross-checks applicant profile against tender eligibility criteria"

  evaluation-criteria-analyzer:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000012"
    type: agent
    description: "Analyzes scoring methodology and award criteria weighting"

  # ── Compliance agents ───────────────────────────────────────────────────────
  compliance-gap-analyzer:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000013"
    type: agent
    description: "Identifies gaps between current ESPD data and tender compliance requirements"

  # ── Financial agents ────────────────────────────────────────────────────────
  financial-analyzer:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000014"
    type: agent
    description: "Analyzes financial health indicators for eligibility scoring"

  risk-assessor:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000015"
    type: agent
    description: "Assesses project implementation risks and mitigation strategies"

  budget-justifier:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000016"
    type: agent
    description: "Generates narrative justification for budget line items"

  # ── Proposal writing agents ─────────────────────────────────────────────────
  proposal-drafter:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000017"
    type: agent
    description: "Drafts full proposal sections from structured project brief"
    timeout_override: 300

  abstract-generator:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000018"
    type: agent
    description: "Generates concise project abstract for tender submission"

  acronym-suggester:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000019"
    type: agent
    description: "Suggests memorable project acronyms from title and objectives"
    timeout_override: 30

  quality-reviewer:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000020"
    type: agent
    description: "Reviews draft proposals for clarity, coherence, and compliance"
    timeout_override: 180

  # ── Team and personnel agents ───────────────────────────────────────────────
  cv-matcher:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000021"
    type: agent
    description: "Matches key expert CVs to tender personnel requirements"

  key-expert-profiler:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000022"
    type: agent
    description: "Generates key expert profiles from CV data and role requirements"

  # ── Project design agents ───────────────────────────────────────────────────
  work-package-designer:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000023"
    type: agent
    description: "Designs work package structure with deliverables and milestones"
    timeout_override: 150

  timeline-planner:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000024"
    type: agent
    description: "Generates Gantt-compatible project timeline from work packages"

  dissemination-planner:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000025"
    type: agent
    description: "Creates dissemination and exploitation plan for funded projects"

  impact-analyzer:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000026"
    type: agent
    description: "Quantifies expected project impact metrics against program objectives"

  sustainability-planner:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000027"
    type: agent
    description: "Develops sustainability strategy for post-grant project continuation"
    timeout_override: 120

  # ── Team (multi-agent collaboration) ───────────────────────────────────────
  proposal-generation-team:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000028"
    type: team
    description: "Coordinated multi-agent team for end-to-end proposal generation"
    timeout_override: 600

  # ── Workflow (sequential pipeline) ─────────────────────────────────────────
  full-proposal-workflow:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000029"
    type: workflow
    description: "Full end-to-end proposal generation workflow from tender to submission-ready document"
    timeout_override: 900
```

### Test Fixture Pattern (Inline YAML in `tmp_path`)

Rather than committing fixture YAML files, use `tmp_path` and inline strings to keep tests self-contained and independent of `config/agents.yaml`:

```python
# tests/unit/test_agent_registry.py
import textwrap
import asyncio
import pytest
from pathlib import Path
from ai_gateway.services.agent_registry import AgentRegistry
from ai_gateway.services.exceptions import AgentNotFoundError

EXEC_UUID = "a1b2c3d4-1234-4abc-8def-000000000001"

VALID_YAML = textwrap.dedent(f"""
agents:
  executive-summary:
    kraftdata_id: "{EXEC_UUID}"
    type: agent
    description: "Executive summary generator"
    timeout_override: 120
  espd-auto-fill:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000002"
    type: agent
    description: "ESPD auto-fill agent"
  grant-eligibility:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000003"
    type: agent
    description: "Grant eligibility checker"
  proposal-generation-team:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000028"
    type: team
    description: "Multi-agent proposal team"
  full-proposal-workflow:
    kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000029"
    type: workflow
    description: "Full proposal pipeline"
""")

@pytest.fixture
async def registry(tmp_path: Path) -> AgentRegistry:
    yaml_file = tmp_path / "agents.yaml"
    yaml_file.write_text(VALID_YAML)
    reg = AgentRegistry(yaml_file)
    await reg.load()
    return reg
```

**Duplicate key test:** Note that because our `_NoDuplicatesLoader` raises on duplicate keys at the YAML parse level, writing a Python string with two `executive-summary:` keys and loading it through `AgentRegistry._parse_yaml` is the correct approach — do not use `yaml.safe_load` in the test itself.

### `services/__init__.py` After This Story

```python
# src/ai_gateway/services/__init__.py — after S04.03
from ai_gateway.services.exceptions import (
    AgentNotFoundError,          # S04.03 addition
    KraftDataAPIError,
    KraftDataConnectionError,
    KraftDataError,
    KraftDataTimeoutError,
)

__all__ = [
    "AgentNotFoundError",
    "KraftDataError",
    "KraftDataTimeoutError",
    "KraftDataConnectionError",
    "KraftDataAPIError",
]
```

### Test Design Mapping (from test-design-epic-04.md)

Relevant test IDs for S04.03:

| Test ID | Priority | Story Test | Key Assertion |
|---|---|---|---|
| **E04-P0-004** | P0 — Critical | `test_load_valid_yaml`, `test_resolve_known_agent`, `test_resolve_unknown_raises` | Registry loads 5 entries; resolve by name returns correct UUID and type; unknown name → `AgentNotFoundError` → HTTP 404 |
| **E04-P1-017** | P1 — High | `test_duplicate_key_raises_on_load` | Duplicate YAML keys → `ValueError` naming the duplicate; service does not start |
| **E04-P2-001** | P2 — Medium | `test_reload_updates_registry` | Hot-reload replaces entries; new agent resolves, old agent raises `AgentNotFoundError` |
| **E04-P2-002** | P2 — Medium | `test_reload_invalid_yaml_keeps_old_registry` | Invalid YAML on reload → `ValueError`; old registry entries still resolve |
| **E04-P2-003** | P2 — Medium | `test_concurrent_resolve_during_reload` | 20 concurrent resolves + 1 reload → all complete without exception or data race |
| **E04-P3-005** | P3 — Low | (integration, S04.10) | `GET /admin/circuits` needs all 29 agents listed — depends on this story's registry wiring |

**Additional tests not in test-design (but required by epic AC):**
- `test_timeout_override_fallback` — validates Pydantic default of `None` for missing optional field
- `test_invalid_uuid_format_raises` — validates UUID v4 regex enforcement
- `test_invalid_type_raises` — validates `Literal["agent", "team", "workflow"]` enforcement
- `test_list_agents_returns_all` — validates admin introspection via `list_agents()`

### Relation to Subsequent Stories

| Story | Dependency on S04.03 |
|---|---|
| S04.04 (sync endpoints) | Calls `registry.resolve(id)` for all `/agents/{id}/run`, `/workflows/{id}/run`, `/teams/{id}/run`; validates `entry.type` matches the endpoint called |
| S04.05 (SSE proxy) | Same resolution pattern as S04.04 for streaming variants |
| S04.06 (circuit breaker) | Circuit state keyed per-agent by `entry.name` (logical name, not UUID) |
| S04.07 (webhook receiver) | Reverse lookup: given KraftData `agent_id` UUID, find `logical_name` by scanning `list_agents()` filtered on `entry.kraftdata_id` |
| S04.08 (execution logging) | Uses `entry.name`, `entry.kraftdata_id`, `entry.type` to populate `gateway.agent_executions` columns |
| S04.09 (rate limiter) | Reads `entry.max_concurrent` for per-agent semaphore cap; `None` means no per-agent limit |
| S04.10 (integration tests) | Loads 5-entry fixture `agents.yaml`; all integration test scenarios use registry-resolved logical names via `resolve()` |

### References

- Epic: `eusolicit-docs/planning-artifacts/epic-04-ai-gateway-service.md` — S04.03 section
- Test design: `eusolicit-docs/test-artifacts/test-design-epic-04.md` — E04-P0-004, E04-P1-017, E04-P2-001, E04-P2-002, E04-P2-003, E04-P3-005
- Risk register (from test design): E04-R-006 (hot-reload race condition), E04-R-009 (YAML UUID drift)
- S04.01 story (scaffold — path conventions): `eusolicit-docs/implementation-artifacts/4-1-fastapi-service-scaffold-and-health-probes.md`
- S04.02 story (KraftData client — existing exceptions pattern): `eusolicit-docs/implementation-artifacts/4-2-httpx-async-client-and-kraftdata-authentication.md`
- Existing exceptions file: `services/ai-gateway/src/ai_gateway/services/exceptions.py`
- Existing services `__init__`: `services/ai-gateway/src/ai_gateway/services/__init__.py`
- Existing config: `services/ai-gateway/src/ai_gateway/config.py`
- Existing main (to extend): `services/ai-gateway/src/ai_gateway/main.py`
- Config dir stub: `services/ai-gateway/config/.gitkeep`
- pyproject.toml (pyyaml already present): `services/ai-gateway/pyproject.toml`
- `eusolicit-kraftdata` response types (for S04.04+): `packages/eusolicit-kraftdata/src/eusolicit_kraftdata/responses.py`

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

- Bug: `_construct_no_dup_mapping` returned `loader.construct_pairs(node)` (a list of pairs) instead of a dict, causing `raw["agents"]` to fail. Fix: wrapped return with `dict(...)`. Root cause was in the story spec's code snippet — the comment said "let pyyaml finish normal construction" but `construct_pairs` does not return a dict.

### Completion Notes List

- ✅ Task 1: `AgentNotFoundError` added to `exceptions.py` with `logical_name` attribute and correct message format. Re-exported from `services/__init__.py` alongside the 4 KraftData exceptions.
- ✅ Task 2: `agent_registry.py` created with `AgentEntry` (Pydantic v2, UUID v4 validator, `Literal` type), `_NoDuplicatesLoader` (YAML loader raising on duplicate keys), `AgentRegistry` class (load/reload/resolve/list_agents/count), and module-level singleton (`init_registry`/`get_registry`). Fixed spec bug: `_construct_no_dup_mapping` must return `dict(loader.construct_pairs(node))`, not the raw pairs list.
- ✅ Task 3: `config/agents.yaml` created with exactly 29 entries (27 agents, 1 team, 1 workflow). Placeholder UUID v4 IDs. `timeout_override` on 10 agents (incl. proposal-drafter: 300, full-proposal-workflow: 900). `max_concurrent: 3` on espd-auto-fill and consortium-finder. `config/.gitkeep` stub deleted.
- ✅ Task 4: `agents_yaml_path: str = "config/agents.yaml"` added to `AIGatewaySettings`.
- ✅ Task 5: `main.py` updated — `init_registry` called in lifespan after `init_client`; `log.info("registry.loaded", count=...)` for E04-R-009 observability; `AgentNotFoundError` exception handler → HTTP 404; admin router registered at `/admin`.
- ✅ Task 6: `routers/admin.py` created — `GET /admin/registry` returns `list[AgentEntry]` sorted by name; `POST /admin/registry/reload` returns `{"loaded": count}` on success or `{"error": "..."}` + HTTP 400 on failure.
- ✅ Task 7: 14 unit tests written covering all ACs and test IDs (E04-P0-004, E04-P1-017, E04-P2-001, E04-P2-002, E04-P2-003). All 14 pass. No regressions in existing 12 tests (26/26 total).
- ✅ Ruff lint: all files clean (3 auto-fixes applied: import ordering in config.py and test file; unused `importlib.resources` import removed from test).

### File List

- `services/ai-gateway/src/ai_gateway/services/exceptions.py` — extended with `AgentNotFoundError`
- `services/ai-gateway/src/ai_gateway/services/__init__.py` — extended to re-export `AgentNotFoundError`
- `services/ai-gateway/src/ai_gateway/services/agent_registry.py` — new file (AgentEntry, AgentRegistry, init_registry, get_registry)
- `services/ai-gateway/config/agents.yaml` — new file (29-entry registry replacing .gitkeep)
- `services/ai-gateway/config/.gitkeep` — deleted
- `services/ai-gateway/src/ai_gateway/config.py` — extended with `agents_yaml_path` field
- `services/ai-gateway/src/ai_gateway/main.py` — extended with registry lifespan wiring, admin router, AgentNotFoundError handler
- `services/ai-gateway/src/ai_gateway/routers/admin.py` — new file (GET /registry, POST /registry/reload)
- `services/ai-gateway/tests/unit/test_agent_registry.py` — new file (14 unit tests)
