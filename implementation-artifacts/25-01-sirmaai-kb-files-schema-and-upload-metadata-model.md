# Story 25.01: SirmaAI KB Files Schema + Upload Metadata Model

**Status:** backlog
**Epic:** E25 — Knowledge Base Lifecycle
**Points:** 3
**Type:** backend
**Dependencies:** S24.01 (sirmaai_projects schema must land first for FK consistency); no dependency on S24.02 (data flow)
**Blocks:** every other E25 story; S24.07 (erasure iterates kb_files)
**Created:** 2026-05-15
**Source:** E25 epic §S25.01

## Story

As a **platform engineer**,
I want **a typed `client.sirmaai_kb_files` table that records EU-Solicit-side metadata pointers to SirmaAI-stored KB artefacts**,
so that **EU Solicit can enforce tier-gated quotas, surface citations, and execute right-to-erasure without holding the artefact bodies themselves (per ADR-019)**.

## Acceptance Criteria

1. Alembic migration in `services/client-api/alembic/versions/` creates `client.sirmaai_kb_files`:
   - `id` UUID PK
   - `company_id` UUID FK → `client.companies(id)` ON DELETE RESTRICT
   - `sirmaai_project_id` UUID FK → `client.sirmaai_projects(id)`
   - `sirmaai_storage_resource_id` TEXT NOT NULL
   - `sirmaai_file_id` TEXT NOT NULL
   - `filename` TEXT NOT NULL
   - `artefact_category` TEXT NOT NULL — StrEnum (`tender|profile|proposal|espd_template|rubric|other`)
   - `tags` JSONB DEFAULT `'[]'::jsonb` (array of strings)
   - `sha256` TEXT NOT NULL (computed during upload streaming)
   - `size_bytes` BIGINT NOT NULL
   - `content_type` TEXT NOT NULL
   - `parsed_text_available_at` TIMESTAMPTZ (null until storage.file.processed webhook)
   - `archived_at` TIMESTAMPTZ (null = active)
   - `archived_reason` TEXT (null = active)
   - `uploaded_by_user_id` UUID FK → `client.users(id)`
   - `created_at`, `updated_at` timestamps
2. **Partial index** `(company_id, artefact_category) WHERE archived_at IS NULL` — supports KB-dashboard filtered queries.
3. UNIQUE constraint on `(company_id, sirmaai_file_id)` — same file_id can be cross-tenant (SirmaAI may reuse IDs); the company scope prevents collisions in EU Solicit.
4. SQLAlchemy model in `services/client-api/src/client_api/models/sirmaai_kb_file.py`:
   - `artefact_category` as `StrEnum` per ADR-012
   - Cross-references the new `SirmaaiProject` model via `relationship`.
5. Service-layer functions in `services/client-api/src/client_api/services/sirmaai_kb_service.py`:
   - `record_uploaded_file(company_id, sirmaai_storage_resource_id, sirmaai_file_id, filename, category, size_bytes, content_type, sha256, user_id) -> SirmaaiKbFile`
   - `mark_processed(sirmaai_file_id) -> None` — sets `parsed_text_available_at = now()`
   - `soft_delete(file_id, reason: str) -> None` — sets `archived_at`
   - `restore(file_id) -> None` — clears `archived_at` (within 30-day window; raises after)
   - `compute_total_size_for_company(company_id) -> int` — sum of size_bytes WHERE archived_at IS NULL (drives tier-quota gate)
   - `list_files_for_company(company_id, category: Optional[str], include_archived=False) -> List[SirmaaiKbFile]`
6. Migration up + down clean on local Postgres.
7. StrEnum validates `artefact_category` at write — invalid value raises `ValueError`.
8. `compute_total_size_for_company` returns sum filtered correctly (verified via unit test).
9. Cross-tenant: tenant A's `list_files_for_company(B)` returns nothing — verified via integration test scoped to client_role.
10. Schema-isolation test (in `tests/integration/test_db_schema_isolation.py`) confirms `client_role` has CRUD on `client.sirmaai_kb_files` and other roles do NOT.

## Dev Notes

### Pattern reuse
- StrEnum: see `services/client-api/src/client_api/models/proposal_version.py`.
- Service-layer pattern: see `services/client-api/src/client_api/services/proposal_service.py`.
- Schema-isolation test pattern: existing in `tests/integration/test_db_schema_isolation.py`.

### Schema (final)
```sql
CREATE TYPE artefact_category AS ENUM ('tender','profile','proposal','espd_template','rubric','other');

CREATE TABLE client.sirmaai_kb_files (
    id                            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    company_id                    UUID NOT NULL REFERENCES client.companies(id) ON DELETE RESTRICT,
    sirmaai_project_id            UUID NOT NULL REFERENCES client.sirmaai_projects(id),
    sirmaai_storage_resource_id   TEXT NOT NULL,
    sirmaai_file_id               TEXT NOT NULL,
    filename                      TEXT NOT NULL,
    artefact_category             artefact_category NOT NULL,
    tags                          JSONB NOT NULL DEFAULT '[]'::jsonb,
    sha256                        TEXT NOT NULL,
    size_bytes                    BIGINT NOT NULL CHECK (size_bytes >= 0),
    content_type                  TEXT NOT NULL,
    parsed_text_available_at      TIMESTAMPTZ,
    archived_at                   TIMESTAMPTZ,
    archived_reason               TEXT,
    uploaded_by_user_id           UUID NOT NULL REFERENCES client.users(id),
    created_at                    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                    TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_kb_files_company_sirmaai_file UNIQUE (company_id, sirmaai_file_id)
);
CREATE INDEX ix_sirmaai_kb_files_active
    ON client.sirmaai_kb_files (company_id, artefact_category)
    WHERE archived_at IS NULL;
```

### Files likely touched
- `services/client-api/alembic/versions/<next>_sirmaai_kb_files.py`
- `services/client-api/src/client_api/models/sirmaai_kb_file.py` (new)
- `services/client-api/src/client_api/services/sirmaai_kb_service.py` (new)
- `services/client-api/src/client_api/models/__init__.py`
- `tests/integration/test_db_schema_isolation.py` (extend)
- `services/client-api/tests/unit/services/test_sirmaai_kb_service.py`

### Out of scope
- Upload endpoint (S25.02)
- Search endpoints (S25.04)
- Replace/archive/restore endpoints (S25.05) — service functions exist, endpoints don't
- Webhook handler (S25.03)

## Testing

- Unit: service-layer happy path + each invariant; StrEnum validation.
- Integration: migration up/down; cross-tenant query isolation; schema-isolation roles.

## See also

- Epic file §S25.01
- Architecture amendment §4.1
- ADR-019 (KB ownership split — SirmaAI canonical, EU Solicit metadata pointers)
- PRD amendment FR-49..FR-52
