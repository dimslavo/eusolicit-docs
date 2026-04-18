# Story 7.1: Proposal & Version DB Schema + Migrations

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **developer on the EU Solicit platform**,
I want **Alembic migration 020 to extend `client.proposals` with E07 proposal-workspace columns and create the new `client.proposal_versions` and `client.content_blocks` tables**,
so that **Stories 7.2–7.10 (Proposal CRUD, Versioning, Content Save, AI Generation, Agent Integrations, Content Blocks, Export) have a fully version-controlled, RLS-ready database schema — with correct enums, indexes, full-text search vector, and a dev seed proposal**.

## Acceptance Criteria

1. **AC1** — Alembic migration `020_proposal_workspace_schema.py` in `services/client-api` runs cleanly (`alembic upgrade head`) against a database with migrations 001–019 applied, and `alembic downgrade -1` (to revision `019_documents`) restores the exact prior schema with no orphaned tables, indexes, or enum types. Re-running `upgrade head` succeeds.

2. **AC2** — After upgrade, `client.proposals` has the following columns (in addition to the columns added by migration 010: `id`, `company_id`, `title`, `milestones`, `budget_summary`, `consortium`, `created_at`, `updated_at`):
   - `opportunity_id UUID` — nullable (soft reference to `pipeline.opportunities`; no DB FK across schema boundaries per project pattern)
   - `status VARCHAR(20) NOT NULL DEFAULT 'draft'` — constrained by CHECK constraint to `('draft', 'active', 'archived')`
   - `current_version_id UUID` — nullable FK to `client.proposal_versions(id) ON DELETE SET NULL` (initially NULL; set after first version is created)
   - `created_by UUID` — nullable soft reference to `client.users.id` (no FK to avoid cascade complexity; application-layer enforced)

3. **AC3** — After upgrade, `client.proposal_versions` exists with:
   - `id UUID PK DEFAULT gen_random_uuid()`
   - `proposal_id UUID NOT NULL FK → client.proposals(id) ON DELETE CASCADE`
   - `version_number INTEGER NOT NULL`
   - `content JSONB NOT NULL DEFAULT '{}'` — stores `{"sections": [{"key": str, "title": str, "body": str}]}`
   - `created_by UUID` — nullable soft reference to user id
   - `change_summary TEXT` — nullable
   - `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
   - UNIQUE constraint on `(proposal_id, version_number)` to enforce sequential, gap-free version ordering per proposal

4. **AC4** — After upgrade, `client.content_blocks` exists with:
   - `id UUID PK DEFAULT gen_random_uuid()`
   - `company_id UUID NOT NULL FK → client.companies(id) ON DELETE CASCADE`
   - `title VARCHAR(500) NOT NULL`
   - `category VARCHAR(100)` — nullable
   - `body TEXT NOT NULL`
   - `tags TEXT[] NOT NULL DEFAULT '{}'`
   - `version INTEGER NOT NULL DEFAULT 1`
   - `approved_by UUID` — nullable (user id, soft reference)
   - `approved_at TIMESTAMPTZ` — nullable
   - `search_vector TSVECTOR` — computed from `title || ' ' || body` via DB trigger; used for GIN full-text search index
   - `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`
   - `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`

5. **AC5** — Required indexes are present after migration:
   - `ix_proposals_opportunity_id` on `client.proposals(opportunity_id)` — supports opportunity-scoped filtering
   - `ix_proposals_status` on `client.proposals(status)` — supports status-filtered listing
   - `ix_proposal_versions_proposal_id` on `client.proposal_versions(proposal_id)` — supports version-list queries
   - `ix_content_blocks_company_id` on `client.content_blocks(company_id)` — required for company-scoped RLS queries
   - `ix_content_blocks_search_vector` GIN index on `client.content_blocks(search_vector)` — enables fast full-text search (S07.09)

6. **AC6** — A DB trigger `content_blocks_search_vector_update` is created on `client.content_blocks` using a PL/pgSQL trigger function `update_content_blocks_search_vector()` that sets `search_vector = to_tsvector('english', coalesce(NEW.title,'') || ' ' || coalesce(NEW.body,''))` on INSERT and UPDATE, ensuring `search_vector` stays fresh after PATCH operations (mitigates E07-R-009).

7. **AC7** — SQLAlchemy ORM models are created/updated:
   - `services/client-api/src/client_api/models/proposal.py` updated: add `opportunity_id`, `status`, `current_version_id`, `created_by` mapped columns; add back-reference to `versions` relationship.
   - `services/client-api/src/client_api/models/proposal_version.py` created: `ProposalVersion` ORM model in `client` schema.
   - `services/client-api/src/client_api/models/content_block.py` created: `ContentBlock` ORM model in `client` schema.
   - `services/client-api/src/client_api/models/__init__.py` updated to export `ProposalVersion` and `ContentBlock`.
   - `alembic check` produces no pending autogenerate changes (ORM ↔ migration in sync).

8. **AC8** — Dev seed script `scripts/seed_sample_proposal.py` inserts one sample proposal (status `draft`) with two `proposal_versions` (version 1 and 2) and three `content_blocks` for a seeded company. Script is idempotent (`INSERT … ON CONFLICT DO NOTHING` or `WHERE NOT EXISTS`). Prints confirmation of inserted vs skipped rows.

9. **AC9** — Integration tests pass: migration runs clean on `eusolicit_test`, all three tables + constraints + indexes + trigger verified, and `downgrade` restores the prior state completely. Tests located at `services/client-api/tests/integration/test_020_migration.py`. Test IDs E07-DB-001 through E07-DB-015 (see Dev Notes).

10. **AC10** — `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py` updated: `ProposalFactory` updated to match E07 schema (adds `current_version_id`, `updated_at`; removes the old `version` field); new `ProposalVersionFactory` and `ContentBlockFactory` added.

## Tasks / Subtasks

- [x] Task 1: Create Alembic migration 020 (AC: 1, 2, 3, 4, 5, 6)
  - [x] 1.1 Create `services/client-api/alembic/versions/020_proposal_workspace_schema.py` with `down_revision = "019_documents"` and `revision = "020_proposal_workspace"`
  - [x] 1.2 `upgrade()` step A — Add E07 columns to `client.proposals`
  - [x] 1.3 `upgrade()` step B — Create `client.proposal_versions` table
  - [x] 1.4 `upgrade()` step C — Add `current_version_id` column to `client.proposals`
  - [x] 1.5 `upgrade()` step D — Create `client.content_blocks` table
  - [x] 1.6 `upgrade()` step E — Create trigger function and BEFORE INSERT OR UPDATE trigger
  - [x] 1.7 `upgrade()` step F — Create all 5 indexes including GIN
  - [x] 1.8 `downgrade()` — reverse in exact reverse order

- [x] Task 2: Update Proposal ORM model (AC: 7)
  - [x] 2.1 Edit `services/client-api/src/client_api/models/proposal.py`

- [x] Task 3: Create ProposalVersion ORM model (AC: 7)
  - [x] 3.1 Create `services/client-api/src/client_api/models/proposal_version.py`

- [x] Task 4: Create ContentBlock ORM model (AC: 7)
  - [x] 4.1 Create `services/client-api/src/client_api/models/content_block.py`

- [x] Task 5: Update models __init__.py (AC: 7)
  - [x] 5.1 Add `from .proposal_version import ProposalVersion`
  - [x] 5.2 Add `from .content_block import ContentBlock`
  - [x] 5.3 Add `ProposalVersion` and `ContentBlock` to `__all__`

- [x] Task 6: Write dev seed script (AC: 8)
  - [x] 6.1–6.8 Create `scripts/seed_sample_proposal.py` with idempotent inserts

- [x] Task 7: Write integration tests — migration 020 (AC: 1, 2, 3, 4, 5, 6, 9)
  - [x] 7.1–7.16 Pre-written ATDD tests in `services/client-api/tests/integration/test_020_migration.py` — all 47 tests pass

- [x] Task 8: Update test-utils factories (AC: 10)
  - [x] 8.1 `ProposalFactory`, `ProposalVersionFactory`, `ContentBlockFactory` in `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py` — already implemented

## Dev Notes

### CRITICAL: Existing Infrastructure — Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/client-api/alembic/versions/001–019*.py` | Existing migration chain | **DO NOT TOUCH** |
| `services/client-api/alembic/env.py` | Configured, schema-aware | **DO NOT TOUCH** |
| `services/client-api/src/client_api/models/proposal.py` | Existing `Proposal` ORM model (E11 reporting columns: milestones, budget_summary, consortium) | **EXTEND ONLY** — add E07 columns; do not remove milestones/budget_summary/consortium |
| `infra/postgres/init/01-init-schemas-and-roles.sql` | Creates 6 schemas + roles | **DO NOT TOUCH** |
| `packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py` | `ALL_SCHEMAS`, `SERVICE_SCHEMA_MAP` | **DO NOT TOUCH** — reuse |

### Why ALTER Instead of Recreate for `proposals`

Migration 010 already created `client.proposals` for Story 11.7 (Logframe Generator — grant/awarded project reporting). That table is live with columns `milestones`, `budget_summary`, `consortium` (grant-project-specific). E07 proposals are *tender response* proposals, which share the same table but need additional workspace columns. Migration 020 **extends** the existing table rather than replacing it:
- Keeps E11 columns (`milestones`, `budget_summary`, `consortium`) untouched
- Adds E07 columns (`opportunity_id`, `status`, `current_version_id`, `created_by`)
- `current_version_id` must be added AFTER `proposal_versions` table is created to satisfy FK constraint

### Cross-Schema Reference Pattern

`proposals.opportunity_id` is a **soft reference** (UUID column, no DB FK) to `pipeline.opportunities`. This is consistent with the project-wide pattern for cross-schema references (see `client.documents.opportunity_id` in migration 019 and E11 `opportunity_compliance_frameworks.opportunity_id`).

`proposals.created_by` is a **soft reference** UUID (no FK to `client.users`) — same pattern as `admin.compliance_frameworks.created_by`. Application-level enforcement in S07.02 router.

### `current_version_id` Circular Reference Handling

`proposals.current_version_id → proposal_versions.id` and `proposal_versions.proposal_id → proposals.id` form a circular FK. In Alembic:
1. Create `proposal_versions` with FK `proposal_id → proposals.id`
2. Add `current_version_id` column to `proposals` (nullable) — FK `→ proposal_versions.id ON DELETE SET NULL`
3. Downgrade reverses: drop `current_version_id` FIRST, then drop `proposal_versions`

```python
# In upgrade():
# Step 1: Create proposal_versions (FK → proposals OK since proposals already exists)
op.create_table("proposal_versions", ...)

# Step 2: Add current_version_id to proposals (FK → proposal_versions now valid)
op.add_column("proposals",
    sa.Column("current_version_id", sa.UUID(as_uuid=True),
              sa.ForeignKey("client.proposal_versions.id", ondelete="SET NULL"),
              nullable=True),
    schema="client")

# In downgrade():
# Step 1: Drop current_version_id FIRST (FK depends on proposal_versions)
op.drop_column("proposals", "current_version_id", schema="client")
# Step 2: Now safe to drop proposal_versions
op.drop_table("proposal_versions", schema="client")
```

### Content Blocks Trigger (PL/pgSQL)

Use `op.execute()` for the trigger function and trigger DDL — Alembic doesn't have a native `create_trigger` op:

```python
# upgrade() — create trigger function:
op.execute("""
    CREATE OR REPLACE FUNCTION update_content_blocks_search_vector()
    RETURNS TRIGGER AS $$
    BEGIN
        NEW.search_vector := to_tsvector(
            'english',
            coalesce(NEW.title, '') || ' ' || coalesce(NEW.body, '')
        );
        RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;
""")

# upgrade() — create trigger:
op.execute("""
    CREATE TRIGGER content_blocks_search_vector_update
    BEFORE INSERT OR UPDATE ON client.content_blocks
    FOR EACH ROW EXECUTE FUNCTION update_content_blocks_search_vector();
""")

# downgrade() — drop trigger then function:
op.execute("DROP TRIGGER IF EXISTS content_blocks_search_vector_update ON client.content_blocks;")
op.execute("DROP FUNCTION IF EXISTS update_content_blocks_search_vector();")
```

### GIN Index on search_vector

```python
op.create_index(
    "ix_content_blocks_search_vector",
    "content_blocks",
    ["search_vector"],
    schema="client",
    postgresql_using="gin",
)
```

### Migration Revision IDs

| Revision | File Name | down_revision |
|----------|-----------|---------------|
| `020_proposal_workspace` | `020_proposal_workspace_schema.py` | `"019_documents"` |

### Files to CREATE (New)

| File | Purpose |
|------|---------|
| `services/client-api/alembic/versions/020_proposal_workspace_schema.py` | Migration extending proposals + creating proposal_versions, content_blocks, trigger, indexes |
| `services/client-api/src/client_api/models/proposal_version.py` | `ProposalVersion` ORM model |
| `services/client-api/src/client_api/models/content_block.py` | `ContentBlock` ORM model |
| `services/client-api/tests/integration/test_020_migration.py` | Integration tests E07-DB-001 through E07-DB-015 |
| `scripts/seed_sample_proposal.py` | Dev seed script — 1 proposal, 2 versions, 3 content blocks |

### Files to MODIFY (Existing)

| File | Change |
|------|--------|
| `services/client-api/src/client_api/models/proposal.py` | Add `opportunity_id`, `status`, `current_version_id`, `created_by` columns; add index entries to `__table_args__`; add `versions` relationship |
| `services/client-api/src/client_api/models/__init__.py` | Export `ProposalVersion`, `ContentBlock` |
| `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py` | Update `ProposalFactory`; add `ProposalVersionFactory`, `ContentBlockFactory` |

### ORM Model Specifications

#### Updated Proposal (`client.proposals`)

```python
class Proposal(Base):
    __tablename__ = "proposals"
    __table_args__ = (
        sa.Index("ix_proposals_company_id", "company_id"),
        sa.Index("ix_proposals_opportunity_id", "opportunity_id"),
        sa.Index("ix_proposals_status", "status"),
        sa.CheckConstraint("status IN ('draft','active','archived')", name="ck_proposals_status"),
        {"schema": "client"},
    )

    id: Mapped[uuid.UUID] = mapped_column(sa.UUID(as_uuid=True), primary_key=True, default=uuid4, server_default=sa.text("gen_random_uuid()"))
    company_id: Mapped[uuid.UUID] = mapped_column(sa.UUID(as_uuid=True), sa.ForeignKey("client.companies.id", ondelete="CASCADE"), nullable=False)
    opportunity_id: Mapped[uuid.UUID | None] = mapped_column(sa.UUID(as_uuid=True), nullable=True)
    title: Mapped[str | None] = mapped_column(sa.String(500), nullable=True)
    status: Mapped[str] = mapped_column(sa.String(20), nullable=False, server_default=sa.text("'draft'"))
    current_version_id: Mapped[uuid.UUID | None] = mapped_column(sa.UUID(as_uuid=True), sa.ForeignKey("client.proposal_versions.id", ondelete="SET NULL"), nullable=True)
    created_by: Mapped[uuid.UUID | None] = mapped_column(sa.UUID(as_uuid=True), nullable=True)
    # Retained from migration 010 for E11 reporting-template compatibility:
    milestones: Mapped[list | None] = mapped_column(JSONB, nullable=True)
    budget_summary: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    consortium: Mapped[list | None] = mapped_column(JSONB, nullable=True)
    created_at: Mapped[datetime] = mapped_column(sa.DateTime(timezone=True), nullable=True, server_default=sa.func.now())
    updated_at: Mapped[datetime] = mapped_column(sa.DateTime(timezone=True), nullable=True, server_default=sa.func.now(), onupdate=sa.func.now())

    versions: Mapped[list["ProposalVersion"]] = relationship(
        "ProposalVersion",
        back_populates="proposal",
        cascade="all, delete-orphan",
        foreign_keys="[ProposalVersion.proposal_id]",
        order_by="ProposalVersion.version_number",
    )
```

#### ProposalVersion (`client.proposal_versions`)

```python
class ProposalVersion(Base):
    __tablename__ = "proposal_versions"
    __table_args__ = (
        sa.UniqueConstraint("proposal_id", "version_number", name="uq_proposal_version_number"),
        sa.Index("ix_proposal_versions_proposal_id", "proposal_id"),
        {"schema": "client"},
    )

    id: Mapped[uuid.UUID] = mapped_column(sa.UUID(as_uuid=True), primary_key=True, default=uuid4, server_default=sa.text("gen_random_uuid()"))
    proposal_id: Mapped[uuid.UUID] = mapped_column(sa.UUID(as_uuid=True), sa.ForeignKey("client.proposals.id", ondelete="CASCADE"), nullable=False)
    version_number: Mapped[int] = mapped_column(sa.Integer, nullable=False)
    content: Mapped[dict] = mapped_column(JSONB, nullable=False, server_default=sa.text("'{}'"))
    created_by: Mapped[uuid.UUID | None] = mapped_column(sa.UUID(as_uuid=True), nullable=True)
    change_summary: Mapped[str | None] = mapped_column(sa.Text, nullable=True)
    created_at: Mapped[datetime] = mapped_column(sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now())

    proposal: Mapped["Proposal"] = relationship(
        "Proposal",
        back_populates="versions",
        foreign_keys=[proposal_id],
    )
```

#### ContentBlock (`client.content_blocks`)

```python
class ContentBlock(Base):
    __tablename__ = "content_blocks"
    __table_args__ = (
        sa.Index("ix_content_blocks_company_id", "company_id"),
        {"schema": "client"},
    )

    id: Mapped[uuid.UUID] = mapped_column(sa.UUID(as_uuid=True), primary_key=True, default=uuid4, server_default=sa.text("gen_random_uuid()"))
    company_id: Mapped[uuid.UUID] = mapped_column(sa.UUID(as_uuid=True), sa.ForeignKey("client.companies.id", ondelete="CASCADE"), nullable=False)
    title: Mapped[str] = mapped_column(sa.String(500), nullable=False)
    category: Mapped[str | None] = mapped_column(sa.String(100), nullable=True)
    body: Mapped[str] = mapped_column(sa.Text, nullable=False)
    tags: Mapped[list] = mapped_column(postgresql.ARRAY(sa.Text), nullable=False, server_default=sa.text("'{}'"))
    version: Mapped[int] = mapped_column(sa.Integer, nullable=False, server_default=sa.text("1"))
    approved_by: Mapped[uuid.UUID | None] = mapped_column(sa.UUID(as_uuid=True), nullable=True)
    approved_at: Mapped[datetime | None] = mapped_column(sa.DateTime(timezone=True), nullable=True)
    # search_vector is managed exclusively by DB trigger; declared as deferred to avoid ORM write attempts
    search_vector: Mapped[Any | None] = mapped_column(TSVECTOR, nullable=True, deferred=True)
    created_at: Mapped[datetime] = mapped_column(sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now())
    updated_at: Mapped[datetime] = mapped_column(sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now(), onupdate=sa.func.now())
```

Note: Import `from sqlalchemy.dialects.postgresql import ARRAY, JSONB, TSVECTOR` and `from typing import Any`.

### Factory Extensions (eusolicit-test-utils)

```python
def ProposalFactory(**overrides: Any) -> dict[str, Any]:
    """Create a tender proposal data dict (E07 schema)."""
    return _merge(
        {
            "id": str(uuid.uuid4()),
            "company_id": str(uuid.uuid4()),
            "opportunity_id": str(uuid.uuid4()),
            "title": fake.sentence(nb_words=6),
            "status": "draft",
            "current_version_id": None,
            "created_by": str(uuid.uuid4()),
            "created_at": datetime.now(UTC).isoformat(),
            "updated_at": datetime.now(UTC).isoformat(),
        },
        overrides,
    )


def ProposalVersionFactory(**overrides: Any) -> dict[str, Any]:
    """Create a proposal version data dict."""
    return _merge(
        {
            "id": str(uuid.uuid4()),
            "proposal_id": str(uuid.uuid4()),
            "version_number": 1,
            "content": {
                "sections": [
                    {"key": "executive_summary", "title": "Executive Summary", "body": fake.paragraph(nb_sentences=3)},
                    {"key": "technical_approach", "title": "Technical Approach", "body": fake.paragraph(nb_sentences=3)},
                ]
            },
            "created_by": str(uuid.uuid4()),
            "change_summary": fake.sentence(nb_words=5),
            "created_at": datetime.now(UTC).isoformat(),
        },
        overrides,
    )


def ContentBlockFactory(**overrides: Any) -> dict[str, Any]:
    """Create a reusable content block data dict."""
    return _merge(
        {
            "id": str(uuid.uuid4()),
            "company_id": str(uuid.uuid4()),
            "title": fake.sentence(nb_words=4),
            "category": fake.random_element(["executive_summary", "technical_approach", "budget_narrative", "legal", "general"]),
            "body": fake.paragraph(nb_sentences=4),
            "tags": [fake.word(), fake.word()],
            "version": 1,
            "approved_by": None,
            "approved_at": None,
            "created_at": datetime.now(UTC).isoformat(),
            "updated_at": datetime.now(UTC).isoformat(),
        },
        overrides,
    )
```

### Test Pattern for Migration Tests

Follow the pattern from `services/client-api/tests/integration/test_002_migration.py` (referenced in the existing 11-1 story):
- Use `TEST_DATABASE_URL` env var (default: `postgresql+asyncpg://migration_role:migration_password@localhost:5432/eusolicit_test`)
- Use `subprocess.run(["alembic", "upgrade", "head"], ...)` for applying migrations
- Use `sqlalchemy.text()` queries against `information_schema.columns`, `pg_indexes`, and `pg_triggers` for verification
- Use `pytest.mark.asyncio` for async test functions
- Use `pytest.fixture` with `yield` + cleanup for test isolation

### Test Coverage Alignment (from test-design-epic-07.md)

This story establishes the database foundation that ALL E07 tests depend on. Key alignment points:

**Entry Criteria Gate (from test-design-epic-07.md):**
- "Database migrations applied: `client.proposals`, `client.proposal_versions`, `client.content_blocks`... tables created with all specified indexes and RLS policies"
- The RLS enforcement (company-scoped) is application-layer in E07 (same pattern as E11); no PostgreSQL RLS policy in this migration — access control is enforced in S07.02 router via JWT `organization_id` claim

**Test IDs directly covered by this story:**
- E07-DB-001: Alembic upgrade succeeds (maps to E07-P3-002 migration reversibility)
- E07-DB-002 through E07-DB-015: Schema structure, constraints, trigger, indexes

**E07 P0 tests that depend on this migration being correct:**
- E07-P0-002 (`generation_status` field) — NOTE: `generation_status` is specified in S07.05 and will be added in a later migration. This story does NOT add it to avoid scope creep.
- E07-P0-003 (concurrent section PATCH optimistic lock) — depends on `proposal_versions.content` JSONB column existing with atomic UPDATE SQL
- E07-P0-005 (RLS cross-company) — depends on `company_id` FK on both `proposals` and `content_blocks` (AC2, AC4)
- E07-P0-006 (content blocks RLS) — depends on `content_blocks.company_id` FK and `search_vector` GIN index

**E07-R-009 mitigation (tsvector stale after PATCH) tested by:**
- E07-DB-008 confirms trigger fires on UPDATE, verifying the AC6 mitigation is in place from day one

**E07-P3-002 (Alembic migration reversibility):**
- Covered by E07-DB-001 and E07-DB-015 in this story's integration tests

### `alembic check` Note

After running `alembic upgrade head`, run `alembic check` to verify ORM models and migrations are in sync. The `search_vector` column's GIN index is created in the migration but NOT declared in the ORM `__table_args__` (since Alembic's autogenerate cannot replicate GIN/trigger-based indexes reliably). To suppress the autogenerate diff for the GIN index and trigger, add the `ix_content_blocks_search_vector` index to the `exclude_tables` or use an `include_object` hook in `env.py` — OR accept the `alembic check` diff for this specific index as a known exception documented in this dev note.

The simplest approach: declare the GIN index in `ContentBlock.__table_args__` using `postgresql_using="gin"`:
```python
sa.Index("ix_content_blocks_search_vector", "search_vector", postgresql_using="gin"),
```
This ensures `alembic check` reports no pending changes.

### Security Notes (E07-R-003)

- `client.proposals` and `client.content_blocks` both have `company_id FK → client.companies(id)` enforcing referential integrity at DB level
- Row-level security is application-enforced via JWT `organization_id` claim in all proposal/content-block endpoints (S07.02, S07.09) — consistent with existing E06 and E11 patterns
- The `opportunity_id` on `proposals` is a soft reference (no FK) — preventing cross-schema FK constraints while still supporting filtering and joins at the application layer
- All cross-company access must return 404 (not 403) per E07-P0-005 spec to avoid ID enumeration

---

## Senior Developer Review

**Reviewer:** BMad Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Date:** 2026-04-17
**Verdict:** APPROVED

### Review History

| Pass | Date | Verdict | Summary |
|------|------|---------|---------|
| 1 | 2026-04-17 | CHANGES REQUESTED | P1 bug: seed script psycopg2 `:name` parameter syntax |
| 2 | 2026-04-17 | **APPROVED** | P1 fix verified; full re-review clean across all 3 layers |

### Pass 2 Summary

All prior findings resolved. Full adversarial re-review (Blind Hunter + Edge Case Hunter + Acceptance Auditor) found zero actionable issues. All 10 acceptance criteria verified against implementation code.

### Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1 | ✅ | Migration 020 has correct revision chain (down_revision=019_documents); upgrade/downgrade ordering handles circular FK correctly |
| AC2 | ✅ | `opportunity_id` (UUID nullable), `status` (VARCHAR(20) NOT NULL DEFAULT 'draft' + CHECK), `current_version_id` (FK SET NULL), `created_by` (UUID nullable) all present |
| AC3 | ✅ | `proposal_versions` table: 7 columns, UNIQUE(proposal_id, version_number), FK CASCADE on proposal_id |
| AC4 | ✅ | `content_blocks` table: 12 columns, correct types (TEXT[], TSVECTOR, JSONB), FK CASCADE on company_id |
| AC5 | ✅ | All 5 indexes created: ix_proposals_opportunity_id, ix_proposals_status, ix_proposal_versions_proposal_id, ix_content_blocks_company_id, ix_content_blocks_search_vector (GIN) |
| AC6 | ✅ | PL/pgSQL trigger function + BEFORE INSERT OR UPDATE trigger on content_blocks; search_vector = to_tsvector('english', title ‖ body) |
| AC7 | ✅ | ORM models created (proposal_version.py, content_block.py), proposal.py extended, __init__.py exports both; GIN index in ContentBlock.__table_args__ for alembic check |
| AC8 | ✅ | Seed script idempotent (WHERE NOT EXISTS + ON CONFLICT DO NOTHING); psycopg2 %(name)s syntax correct; 1 proposal + 2 versions + 3 content blocks |
| AC9 | ✅ | 47 integration tests in test_020_migration.py covering E07-DB-001 through E07-DB-015 plus AC7/AC8/AC10 structural tests |
| AC10 | ✅ | ProposalFactory updated (current_version_id, removed legacy version); ProposalVersionFactory and ContentBlockFactory added |

### Findings (Pass 2)

No actionable findings. 1 stylistic observation dismissed as noise.

#### Resolved from Pass 1

- [x] **[RESOLVED] BUG: Seed script psycopg2 parameter syntax (AC8)** — `:name` syntax replaced with `%(name)s` in content_blocks INSERT and proposals INSERT. Verified correct in current code.

#### Informational (carried forward, no action required)

- **INFO: UNIQUE constraint does not enforce gap-free versioning** — `(proposal_id, version_number)` prevents duplicates but permits gaps. Gap-free enforcement deferred to S07.03 application layer. Consistent with AC3 spec.
- **INFO: Trigger fires on all column updates** — Could add `WHEN (OLD.title IS DISTINCT FROM NEW.title OR OLD.body IS DISTINCT FROM NEW.body)` for efficiency. Current behavior is correct and ensures search_vector is never stale. Optimization deferrable.

### Positive Observations

- **Migration quality is excellent:** Correct circular FK ordering (create proposal_versions → add current_version_id; downgrade: drop current_version_id → drop proposal_versions), clean reversibility, well-documented steps
- **ORM models are accurate:** Types, constraints, relationships, and `__table_args__` all match the migration; search_vector is deferred=True preventing ORM write attempts
- **Tests are comprehensive:** 47 tests covering all 15 E07-DB test IDs plus AC7/AC8/AC10 structural tests; proper fixture teardown restoring DB to head
- **Architecture alignment is strong:** Multi-schema pattern, soft references (cross-schema UUID without FK), factory conventions all consistent with E06/E11 precedent
- **Code documentation is thorough:** Every file has design-decision docstrings and migration-history context

---

## Dev Agent Record

### Implementation Plan

Review continuation session: Addressed P1 code review finding (seed script psycopg2 parameter syntax bug). All original Tasks 1–8 were completed in a prior session.

### Debug Log

- Review finding identified `:name` SQLAlchemy-style parameter syntax in 2 SQL queries within `scripts/seed_sample_proposal.py`
- Fixed content_blocks INSERT (lines 169–176): replaced `:id`, `:company_id`, `:title`, `:category`, `:body`, `:tags` with `%(id)s`, `%(company_id)s`, `%(title)s`, `%(category)s`, `%(body)s`, `%(tags)s`
- Fixed proposals INSERT (lines 202–216): replaced `:id`, `:company_id`, `:title` with `%(id)s`, `%(company_id)s`, `%(title)s`
- Fixed WHERE NOT EXISTS subqueries in both INSERTs: `:title` → `%(title)s`, `:company_id` → `%(company_id)s`
- Verified no remaining `:name` parameter patterns in the file
- proposal_versions INSERTs (lines 233–268) already used correct `%s` positional syntax — no change needed

### Completion Notes

- ✅ Resolved review finding [P1]: Seed script psycopg2 parameter syntax — replaced `:name` with `%(name)s` in 2 SQL queries (content_blocks INSERT and proposals INSERT) plus WHERE NOT EXISTS subqueries
- All 47 integration tests pass (E07-DB-001 through E07-DB-015, AC7, AC8, AC10)
- Full client-api regression suite: 967 passed, 0 new failures (31 pre-existing failures from S7.10/E12, 11 pre-existing errors from E11 migration tests)
- Python syntax validation passed on modified seed script

---

## File List

### New Files (created in prior session)

- `services/client-api/alembic/versions/020_proposal_workspace_schema.py` — Migration 020: extends proposals, creates proposal_versions + content_blocks tables, trigger, indexes
- `services/client-api/src/client_api/models/proposal_version.py` — ProposalVersion ORM model
- `services/client-api/src/client_api/models/content_block.py` — ContentBlock ORM model
- `services/client-api/tests/integration/test_020_migration.py` — Integration tests (47 tests, E07-DB-001–015 + AC7/AC8/AC10)
- `scripts/seed_sample_proposal.py` — Dev seed script (1 proposal, 2 versions, 3 content blocks)

### Modified Files

- `services/client-api/src/client_api/models/proposal.py` — Added E07 columns (opportunity_id, status, current_version_id, created_by), versions relationship, index entries
- `services/client-api/src/client_api/models/__init__.py` — Exports ProposalVersion, ContentBlock
- `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py` — Updated ProposalFactory; added ProposalVersionFactory, ContentBlockFactory
- `scripts/seed_sample_proposal.py` — **[Review fix]** Replaced `:name` parameter syntax with `%(name)s` in content_blocks and proposals INSERT queries

---

## Change Log

- **2026-04-17** — Tasks 1–8 implemented: migration 020, ORM models, seed script, integration tests, factory extensions (all ACs 1–10)
- **2026-04-17** — Addressed code review findings — 1 item resolved: seed script psycopg2 parameter syntax fix (P1 bug)
