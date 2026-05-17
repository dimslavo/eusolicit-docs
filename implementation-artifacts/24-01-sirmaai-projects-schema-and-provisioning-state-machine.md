# Story 24.01: SirmaAI Projects Schema + Provisioning State Machine

**Status:** backlog
**Epic:** E24 — SirmaAI Tenant Provisioning
**Points:** 3
**Type:** backend
**Dependencies:** E04 amendment S04.20 (`sirmaai-gateway` rename) done
**Blocks:** Every other E24 story; E25 S25.01; E26 S26.04; E27/E17 S17.30b
**Created:** 2026-05-15 (via bmad-agent-pm story authoring for orchestrator dispatch)
**Source:** `eusolicit-docs/planning-artifacts/epics/E24-sirmaai-tenant-provisioning.md` §S24.01

## Story

As a **platform engineer**,
I want **canonical Postgres tables and a state machine for SirmaAI Project + MCP-server lifecycle**,
so that **every downstream tenant-provisioning, KB, agent, and CRM story has a typed, audit-traceable substrate to build on**.

## Acceptance Criteria

1. Alembic migration in `services/client-api/alembic/versions/` creates **two `client` schema tables** per architecture amendment §4.1:
   - `client.sirmaai_projects` with columns: `id` (UUID PK), `company_id` (UUID FK → `client.companies`, UNIQUE — one Project per company), `sirmaai_project_id` (TEXT — SirmaAI's identifier), `api_key_encrypted` (BYTEA — Fernet-encrypted Project-scoped API key), `agent_map` (JSONB, default `{}`), `template_version` (TEXT, default `'v1'`), `provisioning_status` (TEXT, NOT NULL — values `pending|provisioned|failed|archived`), `last_error` (TEXT, nullable), `retry_count` (INT, default 0), `archived_at` (TIMESTAMPTZ, nullable), `created_at`, `updated_at`.
   - `client.sirmaai_mcp_servers` with columns: `id` (UUID PK), `company_id` (UUID FK), `provider` (TEXT — `dynamics365|hubspot|...`), `status` (TEXT, NOT NULL — `inactive|registered|error`), `sirmaai_mcp_server_id` (TEXT, nullable until registered), `last_error` (TEXT, nullable), `created_at`, `updated_at`. UNIQUE `(company_id, provider)`.
2. **Partial index** on `client.sirmaai_projects (provisioning_status)` WHERE `provisioning_status <> 'provisioned'` — used by the reconciler (S24.05) to scan non-terminal rows cheaply. Verified via `EXPLAIN ANALYZE` on the reconciler query: index hit, no Seq Scan.
3. SQLAlchemy models in `services/client-api/src/client_api/models/`:
   - `SirmaaiProject` model with `provisioning_status` as a `StrEnum` (`PENDING`, `PROVISIONED`, `FAILED`, `ARCHIVED`) — per ADR-012.
   - `SirmaaiMcpServer` model with `status` as a `StrEnum` (`INACTIVE`, `REGISTERED`, `ERROR`).
4. State-machine helper module `services/client-api/src/client_api/services/sirmaai_provisioning_state.py`:
   - Function `transition(project, new_status)` enforces valid transitions: `pending → provisioned | failed`, `provisioned → archived`, `failed → pending` (retry), `archived` (terminal).
   - Invalid transition raises `InvalidProvisioningTransition`.
   - Every transition writes a `shared.audit_log` row with `action_type='sirmaai_project_state_transition'`, `before`, `after`.
5. Migration downgrade cleanly drops both tables + index + enum types; verified on local Postgres.
6. UNIQUE constraint on `client.sirmaai_mcp_servers (company_id, provider)` enforced; duplicate insert raises `IntegrityError`.
7. Unit tests cover all valid + invalid state transitions (≥15 cases); pytest mark `@pytest.mark.unit`.
8. Integration test runs the migration on a fresh database via `make reset-db && make migrate-service SVC=client-api`; tables visible via `\dt client.*`; both StrEnum types visible via `\dT client.*`.

## Dev Notes

### Pattern reuse (canonical)
- Alembic migration scaffolding: copy structure from existing `services/client-api/alembic/versions/074_*.py` style (most recent migration).
- StrEnum pattern: see `services/client-api/src/client_api/models/proposal_version.py` — same shape.
- Audit-log write helper: `client_api.services.audit_service.log_mutation()` — already exists, reuse.

### Schema migration ordering
- This story's migration MUST run before any E25/E26 migrations that reference these tables (FK constraints will fail otherwise).
- Use `down_revision` chained to the latest existing client-api revision.

### Schema reminder
```sql
CREATE TYPE sirmaai_project_status AS ENUM ('pending','provisioned','failed','archived');
CREATE TYPE sirmaai_mcp_server_status AS ENUM ('inactive','registered','error');

CREATE TABLE client.sirmaai_projects (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id              UUID NOT NULL UNIQUE REFERENCES client.companies(id) ON DELETE RESTRICT,
    sirmaai_project_id      TEXT,
    api_key_encrypted       BYTEA,
    agent_map               JSONB NOT NULL DEFAULT '{}'::jsonb,
    template_version        TEXT NOT NULL DEFAULT 'v1',
    provisioning_status     sirmaai_project_status NOT NULL DEFAULT 'pending',
    last_error              TEXT,
    retry_count             INT NOT NULL DEFAULT 0,
    archived_at             TIMESTAMPTZ,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_sirmaai_projects_nonterminal
    ON client.sirmaai_projects(provisioning_status)
    WHERE provisioning_status <> 'provisioned';

CREATE TABLE client.sirmaai_mcp_servers (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id              UUID NOT NULL REFERENCES client.companies(id) ON DELETE RESTRICT,
    provider                TEXT NOT NULL,
    status                  sirmaai_mcp_server_status NOT NULL DEFAULT 'inactive',
    sirmaai_mcp_server_id   TEXT,
    last_error              TEXT,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_sirmaai_mcp_servers_company_provider UNIQUE (company_id, provider)
);
```

### Files likely touched
- `services/client-api/alembic/versions/<next>_sirmaai_projects_and_mcp_servers.py`
- `services/client-api/src/client_api/models/sirmaai_project.py` (new)
- `services/client-api/src/client_api/models/sirmaai_mcp_server.py` (new)
- `services/client-api/src/client_api/services/sirmaai_provisioning_state.py` (new)
- `services/client-api/src/client_api/models/__init__.py` (re-export)
- `services/client-api/tests/unit/services/test_sirmaai_provisioning_state.py` (new)
- `tests/integration/test_db_schema_isolation.py` (extend to cover new tables under `client_role` CRUD)

### Out of scope
- Provisioning task implementation (that's S24.02)
- API key encryption helper module (use existing Fernet canonical from Epic 9)
- Frontend / admin UI (S24.05)

## Risks

- **R1**: StrEnum vs native PG enum drift — if a value is added in code without a migration, runtime errors result. Lock test that asserts StrEnum members match PG enum values.
- **R2**: `company_id` FK with `ON DELETE RESTRICT` means an attempted company delete fails if a Project row exists — by design (forces archival via S24.06 first).

## Testing

- Unit: state-machine transitions (valid + invalid), StrEnum value lock test.
- Integration: migration up/down clean; `EXPLAIN ANALYZE` reconciler-style query uses partial index (no Seq Scan).
- Cross-tenant negative: not applicable yet (no API endpoint); covered downstream.

## See also

- Epic file: `eusolicit-docs/planning-artifacts/epics/E24-sirmaai-tenant-provisioning.md` §S24.01
- Architecture amendment: `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` §4.1
- PRD amendment: FR-45 (provisioning) + FR-46 (archival)
- Schema isolation rule: `eusolicit-app/CLAUDE.md` §DB schema isolation
