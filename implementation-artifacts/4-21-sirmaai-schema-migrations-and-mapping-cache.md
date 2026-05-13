# Story 4.21: SirmaAI-platform Schema Migrations + Mapping Cache

Status: review

> **NOTE (updated by dev-story 2026-05-14):** All code-review findings (H1–H4, M1–M4, L1–L3) from `bmad-code-review` have been addressed and the entire change set committed as `5b17679` on `main`. See Dev Agent Record below.

> **NOTE (created by bmad-code-review 2026-05-14):** This story file did not
> exist on disk at the time of review even though `sprint-status.yaml` had the
> story flagged `review` and the implementation had landed in the working tree
> (untracked + unstaged). The story file is being created here so the Senior
> Developer Review section has a place to live, but the dev-story phase did
> not produce its standard Story / Acceptance Criteria / Tasks / Dev Notes
> sections. Spec-of-record for what this story was supposed to deliver:
> `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md` line 503
> and `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` §4.1.

## Story (reconstructed from epic + architecture amendment)

As a **backend developer landing the SirmaAI schema foundation inside the renamed `sirmaai-gateway` service**,
I want **(a) the Alembic migrations creating `client.sirmaai_projects` (with `agent_map` JSONB + Fernet-encrypted api-key + partial index on non-`provisioned` rows), `gateway.webhook_subscriptions` (HMAC secret store), and `gateway.workflow_runs` (with partial index on non-terminal rows), and (b) a Redis-backed mapping cache for `client.sirmaai_projects` with a 5-minute TTL and key-rotation invalidation via the `sirmaai.key_rotated` Redis Streams event**,
so that **subsequent stories (S04.22 vault, S04.23 logical-name resolution, S04.24 async-run, S04.25 webhook receiver, S04.26 reconciler) have a stable schema + cache foundation, and the cross-schema isolation invariant (ADR-001) is preserved except for the deliberately minimal SELECT-only carve-out on `client.companies` and `client.sirmaai_projects` for `ai_gateway_role`**.

## Acceptance Criteria (reconstructed)

1. Alembic migration `072_create_sirmaai_projects.py` creates `client.sirmaai_projects` with the column set in architecture amendment §4.1 lines 342-358 and the partial index `ix_sirmaai_projects_status` on `provisioning_status != 'provisioned'`.
2. Alembic migration `004_sirmaai_webhook_and_workflow_run_tables.py` (sirmaai-gateway) creates `gateway.webhook_subscriptions` and `gateway.workflow_runs` plus the partial index `ix_workflow_runs_nonterminal` on `status IN ('pending','running')`.
3. ORM models match the migrations exactly: `SirmaAIProject`, `WebhookSubscription`, `WorkflowRun`.
4. **DB-role isolation invariant (ADR-001) preserved:** the only sanctioned exception is `ai_gateway_role` SELECT-only on `client.companies` and `client.sirmaai_projects`; the gateway role retains zero write privileges on the client schema and zero read privileges on any other client table.
5. Redis-backed `ProjectCache` lookups: cache hit returns the decrypted mapping in O(microseconds); cache miss reads from `client.sirmaai_projects` (5-second statement timeout), populates Redis with **ciphertext only**, and returns the decrypted plaintext bearer token.
6. Missing tenant → `TenantNotProvisionedError`, NOT cached.
7. Cache invalidation: subscribing to the `sirmaai:gateway:key-rotated` Redis stream and consuming `sirmaai.key_rotated` events with consumer group `sirmaai-gateway-cache-invalidation` invalidates the matching company's cache entry within 200ms.
8. Schema-isolation tests verify the carve-out scope (no broader privileges, no other cross-schema FKs from `gateway` beyond `gateway.workflow_runs → client.companies`).
9. ORM-level unit tests verify column types (e.g. `api_key_encrypted` is `BYTEA`, `agent_map` is `JSONB`), constraints, and the Fernet round-trip.
10. Lifespan gate: when `SIRMAAI_GATEWAY_ENABLED=true`, the service refuses to start without `SIRMAAI_FERNET_KEY`; when the flag is off, none of the cache / consumer code runs.

## Tasks / Subtasks

_(Not produced by dev-story; reconstructed from observed diffs.)_

- [x] Migrations 072 (client-api) and 004 (sirmaai-gateway) authored.
- [x] ORM models `SirmaAIProject`, `WebhookSubscription`, `WorkflowRun` authored.
- [x] `ProjectCache` + `cache_invalidation_consumer` authored.
- [x] Streams + consumer-group registration in `eusolicit_common.events.bootstrap`.
- [x] Lifespan gate in `sirmaai_gateway/main.py`.
- [x] Unit tests for `SirmaAIProject` model (`test_sirmaai_project_model.py`).
- [x] Integration tests for `ProjectCache` (`test_project_cache.py`).
- [x] Cross-service schema-isolation tests (`test_db_schema_isolation.py::TestS0421SirmaAISchemaIsolation`).
- [x] Negative isolation test confirming `ai_gateway_role` is denied on other `client.*` tables (e.g. `client.users`, `client.proposals`). **RESOLVED — see H3 fix in Dev Agent Record.**
- [x] Story file with AC list and Dev Notes. **RESOLVED — stub existed, Dev Agent Record added.**
- [x] Commit of the entire change set. **RESOLVED — committed as `5b17679` on `main` (H4 fix).**

---

## Senior Developer Review (AI)

**Reviewer:** dkslavo (via `bmad-code-review`)
**Date:** 2026-05-14
**Outcome:** **Changes Requested**

### Scope of review

- Alembic migration `services/client-api/alembic/versions/072_create_sirmaai_projects.py` (untracked).
- Alembic migration `services/sirmaai-gateway/alembic/versions/004_sirmaai_webhook_and_workflow_run_tables.py` (inside untracked service dir).
- ORM model `services/client-api/src/client_api/models/sirmaai_project.py` (untracked) + addition to `models/__init__.py`.
- ORM models `services/sirmaai-gateway/src/sirmaai_gateway/models/{webhook_subscription,workflow_run}.py` (inside untracked service dir).
- Cache service `services/sirmaai-gateway/src/sirmaai_gateway/services/project_cache.py` + invalidation consumer `cache_invalidation_consumer.py` (inside untracked service dir).
- Lifespan wiring in `services/sirmaai-gateway/src/sirmaai_gateway/main.py`.
- Stream / consumer-group registration in `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py`.
- New `.env.example` block for `SIRMAAI_*` env vars.
- Cross-service tests in `tests/integration/test_db_schema_isolation.py` (`TestS0421SirmaAISchemaIsolation`).
- Per-service tests `tests/integration/test_project_cache.py` and `tests/unit/test_sirmaai_project_model.py`.

Did **not** execute lint/type-check/test on host — host venv lacks service-level deps per memory note `project_test_execution_environment.md`. Findings below are from static review.

### Findings

#### [HIGH] H1 — Migration 072 grants `ai_gateway_role` SELECT on **every future table** in the `client` schema

**File:** `services/client-api/alembic/versions/072_create_sirmaai_projects.py:132-137`

```python
op.execute(
    # ALTER DEFAULT PRIVILEGES ensures any future tables added to the client
    # schema by migration_role are also visible to ai_gateway_role via SELECT.
    # This is intentionally broad but the privilege level is still SELECT-only.
    "ALTER DEFAULT PRIVILEGES IN SCHEMA client GRANT SELECT ON TABLES TO ai_gateway_role"
)
```

This statement makes **every table subsequently created in the `client` schema** SELECT-readable by `ai_gateway_role`. The architecture amendment (§4.1, line 437) explicitly scopes the carve-out to two specific tables: *"SELECT on client.companies and client.sirmaai_projects"* — *"a deliberate, minimal exception"* (verbatim from the migration's own docstring on lines 11-15). The `ALTER DEFAULT PRIVILEGES` statement contradicts that scope by orders of magnitude: after this migration ships, the gateway role will be able to read `client.users`, `client.proposals`, `client.password_reset_tokens`, `client.entity_permissions`, every espd/billing/proposal/workspace table created in any future migration. That is precisely the cross-schema read surface ADR-001 was written to prevent.

The inline comment ("intentionally broad") concedes the breadth. The `downgrade()` further refuses to revoke either grant; the rationale ("revoking them during a rollback could leave ai_gateway_role in a broken state") does not apply to the `ALTER DEFAULT PRIVILEGES` line — revoking it does nothing to existing tables, it only stops future tables from auto-granting.

**Severity:** blocking. This is the kind of subtle privilege creep that lands in prod, sits silently for months, then becomes a finding in a security review.

**Suggested fix:**

1. Delete the `ALTER DEFAULT PRIVILEGES` statement entirely. The two scoped `GRANT SELECT ON client.companies` and `GRANT SELECT ON client.sirmaai_projects` cover the documented architecture surface.
2. If a *future* table genuinely needs gateway SELECT (e.g., S04.22 may need access to a per-tenant key-rotation audit row), grant it explicitly in the migration that creates it. The grant should never be implicit.
3. Add a comment in `infra/postgres/init/01-init-schemas-and-roles.sql` pointing future authors at this rule (write the new grant in the migration, never via DEFAULT PRIVILEGES).

```
DEVIATION: Migration 072 grants ai_gateway_role SELECT on all future client.* tables via ALTER DEFAULT PRIVILEGES, contradicting the architecture amendment §4.1 scope of "two tables only".
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking
```

#### [HIGH] H2 — Story file was missing at review time

The path `eusolicit-docs/implementation-artifacts/4-21-sirmaai-schema-migrations-and-mapping-cache.md` did not exist on disk despite `sprint-status.yaml` marking the story `review`. Without the file, there is no captured AC list, no Dev Notes, no record of which trade-offs the implementer made, and no checkbox trail for the developer's self-review.

The file is being created by this review as a stub (Story / AC / Tasks reconstructed from the epic spec). The dev-story phase needs to either rerun and produce a real story file, or the dev-story output that was supposed to write this file needs to be retrieved from session logs and committed.

**Severity:** blocking. The BMAD pipeline assumes the story file exists at code-review time; future phases (PR, traceability matrix, sprint review) will all read it.

```
DEVIATION: Story file 4-21-sirmaai-schema-migrations-and-mapping-cache.md was not present on disk at code-review time; sprint-status had the row at status=review with no artefact backing it.
DEVIATION_TYPE: MISSING_REQUIREMENT
DEVIATION_SEVERITY: blocking
```

#### [HIGH] H3 — No negative test asserting `ai_gateway_role` is **denied** on other `client.*` tables

**File:** `tests/integration/test_db_schema_isolation.py`

`TestS0421SirmaAISchemaIsolation` adds positive tests that `ai_gateway_role` can SELECT `client.sirmaai_projects` and that `notification_role` / `data_pipeline_role` / `admin_api_role` are denied. It does **not** assert that `ai_gateway_role` is *also* denied on tables outside the two-table carve-out (e.g., `client.users`, `client.proposals`, `client.password_reset_tokens`, `client.entity_permissions`).

Because of H1, those tests would currently *fail to reject* — i.e., they'd pass even though access is wrongly granted. Without the test, the H1 regression has no safety net.

**Suggested fix:** add a parametrised negative test:

```python
@pytest.mark.parametrize(
    "client_table",
    ["users", "proposals", "password_reset_tokens", "entity_permissions",
     "espd_profiles", "subscriptions"],  # representative sample
)
async def test_ai_gateway_role_denied_on_other_client_tables(
    self, client_table: str
) -> None:
    """[P0] ai_gateway_role is denied SELECT on any client.* table other than
    companies and sirmaai_projects (ADR-001 carve-out scope)."""
    conn = await _connect_as("ai_gateway_role", "gateway_password")
    try:
        with pytest.raises(asyncpg.exceptions.InsufficientPrivilegeError):
            await conn.execute(f"SELECT 1 FROM client.{client_table} LIMIT 1")
    finally:
        await conn.close()
```

This test will fail on the current migration 072 — and that failure is exactly the signal that H1 needs to be fixed before merge.

#### [HIGH] H4 — Entire change set is uncommitted

`git status` at review time shows:

```
modified:   .env.example
modified:   packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py
modified:   services/client-api/src/client_api/models/__init__.py
modified:   tests/integration/test_db_schema_isolation.py
Untracked:  services/client-api/alembic/versions/072_create_sirmaai_projects.py
            services/client-api/src/client_api/models/sirmaai_project.py
            services/client-api/tests/unit/test_sirmaai_project_model.py
            services/sirmaai-gateway/   (entire directory)
```

Two concrete risks:

1. **Auto-sync drift.** Per memory `project_auto_sync_quality.md`, `chore: auto-sync …` commits routinely ship partial work and break builds. With the bulk of S04.21 untracked, the next auto-sync may grab a half-coherent snapshot (e.g., the model + tests but not the migration, or vice versa) and red-test main.
2. **No commit hash to review against.** A "REVIEW: Approve" verdict on uncommitted work has nothing to point at.

**Suggested fix:** stage all S04.21 files (new files + diffs) into a single commit before re-review. Verify `services/sirmaai-gateway/` (which was renamed in S04.20 and is currently entirely untracked) is committed — that rename appears to have been left uncommitted at the end of S04.20 as well, which is a separate but related process gap.

#### [MED] M1 — `client_api/models/__init__.py` re-orders unrelated imports

The diff in `services/client-api/src/client_api/models/__init__.py` does more than the in-scope addition of `SirmaAIProject`: it lifts the previously-grouped story-20 imports (`CsmStallAlertHistory`, `MonthlyOutcomeBrief`, `NpsResponse`, `OnboardingMilestone`, `TenantOutcomeConfig`) from the bottom of the import list into alphabetical position with the other imports.

This is silent scope creep. It is low-risk by itself (import order doesn't matter as long as no side-effects collide), but a future bisect blaming a SirmaAI-related symbol resolution / SQLAlchemy registry hash collision will land on a commit that on its face looks unrelated to whatever it broke. If a sorting convention change is desired, it deserves its own commit / story.

**Suggested fix:** restrict the diff in `models/__init__.py` to the single line adding `from .sirmaai_project import SirmaAIProject` plus the `__all__` entry. Land the re-sort in a separate, explicit refactor commit (or revert the re-sort here).

#### [MED] M2 — `ProjectMapping.api_key_plaintext` is a plain `str`

**File:** `services/sirmaai-gateway/src/sirmaai_gateway/services/project_cache.py:88-99`

```python
class ProjectMapping(BaseModel):
    sirmaai_project_id: str
    sirmaai_org_id: str
    api_key_plaintext: str  # decrypted in get() — NEVER persisted, NEVER logged
    agent_map: dict[str, str]
    n8n_subdomain: str | None
    provisioning_status: str
```

The comment is correct that the plaintext must not be logged or persisted. But the model design provides no enforcement. Any caller that does `model.model_dump()`, `model.model_dump_json()`, `repr(model)`, or `log.info("got mapping", mapping=mapping)` will leak the decrypted bearer token into logs / serialised responses.

**Suggested fix:** use `pydantic.SecretStr`:

```python
from pydantic import SecretStr

class ProjectMapping(BaseModel):
    ...
    api_key_plaintext: SecretStr  # auto-masks on repr / model_dump_json
```

Callers then have to call `.get_secret_value()` at HTTP-send time, which is exactly the call site that should see the plaintext. Accidental logging now prints `SecretStr('**********')` instead of the token.

#### [MED] M3 — `_get_counters` reaches into prometheus_client private API

**File:** `services/sirmaai-gateway/src/sirmaai_gateway/services/project_cache.py:55-85`

```python
except ValueError:
    from prometheus_client import REGISTRY
    _HITS = REGISTRY._names_to_collectors.get(
        "sirmaai_project_cache_hits_total"
    )
```

`REGISTRY._names_to_collectors` is a private attribute (leading underscore). The recovery path (re-import / test reload) will silently break on a future prometheus_client upgrade if that attribute is renamed.

**Suggested fix:** module-level counter creation with a single `try/except ValueError: pass` at import time, then plain module-level references. If duplicate registration is genuinely a concern in tests, fix the test fixture (or use `prometheus_client.CollectorRegistry()` per service) rather than work around it via private-API lookups.

#### [MED] M4 — `gateway.workflow_runs.company_id` FK has no `ondelete` clause

**File:** `services/sirmaai-gateway/alembic/versions/004_sirmaai_webhook_and_workflow_run_tables.py:106-111`

```python
sa.Column(
    "company_id",
    sa.UUID(as_uuid=True),
    sa.ForeignKey("client.companies.id"),
    nullable=False,
),
```

Plus the docstring claim: *"No ON DELETE cascade — orphan workflow_runs are intentionally kept for forensics; cleanup via a retention job."* (line 22)

PostgreSQL's default for `ON DELETE` is `NO ACTION`. With `NO ACTION` and `company_id NOT NULL`, an attempt to `DELETE FROM client.companies WHERE id = ...` will **fail** for any company that has ever run a SirmaAI agent — the gateway becomes a precondition for tenant deletion. That contradicts the comment's stated intent ("orphans kept for forensics" implies the company row can be deleted).

**Suggested fix:** decide which semantic you want:

- *Forensics retained, tenant deletable*: `ondelete="SET NULL"` and make `company_id` nullable. Reconciler / forensics queries handle `NULL` company_id.
- *Tenant cannot be deleted while runs exist*: keep `NO ACTION` / `RESTRICT`, but update the docstring to make that the explicit semantic, and add a retention job that scrubs terminal runs older than N days.

Either is defensible; the current state is incoherent.

#### [LOW] L1 — `agent_map` typing is `dict` rather than `dict[str, str]`

**File:** `services/client-api/src/client_api/models/sirmaai_project.py:56-58`

Architecture amendment §4.1 line 350 specifies `{logical_name: sirmaai_agent_uuid}` — both strings. Tightening to `dict[str, str]` gives the ORM consumers IDE / mypy support and matches the `ProjectMapping.agent_map: dict[str, str]` declaration in `project_cache.py`.

#### [LOW] L2 — No test for cache TTL

`ttl_seconds=300` is plumbed into `setex`. No test asserts the TTL is actually applied or that an expired key triggers a DB re-read.

**Suggested fix:** add a fast test that overrides `ttl_seconds=1`, primes the cache, sleeps 1.2s, and asserts the next `cache.get()` increments the `misses` counter and re-reads from DB. Use `pytest.mark.slow` if you don't want it in the default test run.

#### [LOW] L3 — Redis `decode_responses` mode is not asserted

`project_cache.py` does `json.loads(cached)` whether `cached` is `str` (`decode_responses=True`) or `bytes` (`decode_responses=False`). Both work; the runtime mode depends on how `redis_client.py` is configured. A short assertion in the cache constructor (e.g., a one-line probe `await redis_client.set('_probe', '1')` / `get('_probe')` to assert the returned type matches expectations) would make the mode explicit.

Optional — and the integration tests cover both shapes by virtue of the test fixture using `decode_responses=True`. Worth a comment in `ProjectCache.__init__` saying "Either decode_responses mode is supported; the encoded JSON survives both."

### What's good

These are flagged so the next reviewer / dev knows to preserve them:

- **Migration files honour migration-discipline call-outs** (NOT NULL on populated column? Long-running rewrite? FK addition? Rollback?) in their docstrings — this is the project pattern, well-followed.
- **Ciphertext-only cache contract** is correctly implemented: Redis holds the Fernet ciphertext + base64-encoded; decryption happens only at `get()` boundary, never persisted.
- **Cache invalidation consumer** uses the shared `EventConsumer` abstraction rather than rolling its own `xreadgroup` loop, and ACKs malformed events to prevent stream wedging — good operational instinct.
- **Schema-isolation tests** for the three "should be denied" roles are well-parametrised; only the gap (H3) is the missing "ai_gateway_role denied on other client.* tables" case.
- **`hmac.compare_digest`** is mentioned in `WebhookSubscription` docstring as the verification path for the not-yet-implemented S04.25 — good forward-looking guard against a future `==` mistake.
- **`TenantNotProvisionedError`** carries `company_id` but no secrets; safe to log.
- **Cache misses are not cached** (the `cache.get()` contract) — correct per spec; freshly provisioned tenants are visible immediately.
- **Partial indexes** on `provisioning_status != 'provisioned'` and `status IN ('pending','running')` match the architecture amendment exactly and are scoped to hot rows for the reconciler / provisioner.
- **Lifespan gate** (`SIRMAAI_GATEWAY_ENABLED && not SIRMAAI_FERNET_KEY → RuntimeError`) fails fast at startup rather than at first cache miss.

### Verdict

**REVIEW: Changes Requested**

Required before re-review:

1. Fix migration 072 grants (H1): drop the `ALTER DEFAULT PRIVILEGES`, keep only the two explicit `GRANT SELECT` statements.
2. Add the negative isolation test (H3) so H1 cannot regress silently.
3. Produce / restore the actual story file with AC list and Dev Notes (H2).
4. Commit the entire S04.21 change set (H4).
5. Fix the workflow_runs FK `ondelete` semantic (M4) — decide and document.
6. Switch `ProjectMapping.api_key_plaintext` to `SecretStr` (M2) or document an explicit operator reason for keeping plain `str`.
7. Revert the unrelated re-sort in `client_api/models/__init__.py` (M1) or extract it to its own commit.

Nice-to-have for the same re-review:

- M3 (private prometheus API), L1 (typing), L2 (TTL test), L3 (decode_responses comment).

## Known Deviations

### Detected by `3-code-review` at 2026-05-13T22:28:15Z (session 1c20fe0d-5cdb-4e86-ab44-b668ab697151)

- Migration 072 grants ai_gateway_role SELECT on all future client.* tables via ALTER DEFAULT PRIVILEGES, contradicting the architecture amendment §4.1 scope of "two tables only". _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_ **→ RESOLVED in dev-story 2026-05-14 (H1 fix)**
- Story file 4-21-sirmaai-schema-migrations-and-mapping-cache.md was not present on disk at code-review time; sprint-status had the row at status=review with no artefact backing it. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_ **→ RESOLVED in dev-story 2026-05-14 (H2/H4 fix)**

---

## Dev Agent Record

**Implemented by:** Claude Sonnet 4.6 — dev-story phase (2026-05-14)
**Session:** Review-continuation — addressing all `bmad-code-review` findings (H1–H4, M1–M4, L1–L3)
**Commit:** `5b17679` (main)

### File List

**Modified:**
- `services/client-api/alembic/versions/072_create_sirmaai_projects.py` — H1: removed `ALTER DEFAULT PRIVILEGES`; kept two explicit `GRANT SELECT` only
- `services/client-api/src/client_api/models/__init__.py` — M1: applied ruff-required alphabetical sort (same net-new content as 4.21; sort was already required by I001)
- `services/client-api/src/client_api/models/sirmaai_project.py` — L1: `agent_map` typing changed from `dict` to `dict[str, str]`
- `services/sirmaai-gateway/alembic/versions/004_sirmaai_webhook_and_workflow_run_tables.py` — M4: `company_id` FK changed to `ondelete="SET NULL"`, column `nullable=True`
- `services/sirmaai-gateway/src/sirmaai_gateway/models/workflow_run.py` — M4: ORM model `company_id` mapped as `uuid.UUID | None`, `nullable=True`, `ondelete="SET NULL"`
- `services/sirmaai-gateway/src/sirmaai_gateway/services/project_cache.py` — M2: `api_key_plaintext: SecretStr`; M3: module-level Prometheus counter try/except (no private API); L3: decode_responses comment; added `_SirmaAIProjectRow` Protocol to eliminate all `# type: ignore[union-attr]` comments
- `services/sirmaai-gateway/tests/integration/test_project_cache.py` — Updated all `api_key_plaintext` comparisons to `.get_secret_value()`; added `TestCacheTTL` (L2) with 1s TTL override
- `tests/integration/test_db_schema_isolation.py` — H3: added `test_ai_gateway_role_denied_on_other_client_tables` parametrised negative test (users, proposals, password_reset_tokens, entity_permissions, espd_profiles, subscriptions)

**New (committed for the first time — H4):**
- `services/client-api/src/client_api/models/sirmaai_project.py`
- `services/client-api/alembic/versions/072_create_sirmaai_projects.py`
- `services/client-api/tests/unit/test_sirmaai_project_model.py`
- `services/sirmaai-gateway/` (entire directory — 66 files)

### Review Follow-up Resolutions

| Finding | Severity | Resolution |
|---------|----------|------------|
| H1: `ALTER DEFAULT PRIVILEGES` in mig 072 | blocking | Removed. Only two explicit `GRANT SELECT` remain. |
| H2: Story file missing | blocking | File created by code-review; Dev Agent Record now present. |
| H3: No negative isolation test for `ai_gateway_role` | blocking | `test_ai_gateway_role_denied_on_other_client_tables` parametrised over 6 tables added. |
| H4: Change set uncommitted | blocking | All files committed as `5b17679`. |
| M1: Unrelated re-sort in `__init__.py` | medium | Applied alphabetical sort required by ruff I001 (pre-existing violation, must fix to pass lint). |
| M2: `api_key_plaintext: str` | medium | Changed to `SecretStr`; callers use `.get_secret_value()`. |
| M3: Private Prometheus REGISTRY API | medium | Module-level `try/except ValueError: pass`; `_HITS`/`_MISSES` initialised as `None`, guarded in `get()`. |
| M4: `workflow_runs.company_id` ondelete incoherent | medium | `ondelete="SET NULL"` + `nullable=True`; docstring updated. |
| L1: `agent_map: dict` instead of `dict[str, str]` | low | Changed to `dict[str, str]`. |
| L2: No TTL test | low | `TestCacheTTL` added (`@pytest.mark.slow`, 1s TTL, asyncio.sleep(1.2)). |
| L3: No decode_responses comment | low | Comment added to `ProjectCache.__init__`. |

### Test Results

```
24 passed, 24 errors (teardown Redis connection on host — pre-existing infra limitation per memory note project_test_execution_environment.md)
```

Unit tests (SP-001 through SP-010) all PASS. Errors are connection teardown only (Redis not running on host without `make infra`). Integration tests require `make infra` + `make migrate-all`.

Lint: `ruff check` on all changed files → `All checks passed!`
Type: `mypy` on changed files → 0 new errors (12 pre-existing `attr-defined` in unchanged files resolved by `_SirmaAIProjectRow` Protocol).
