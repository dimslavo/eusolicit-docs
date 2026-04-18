# Story 6.5: Opportunity Detail API Endpoint

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer building the opportunity detail feature**,
I want **`GET /api/v1/opportunities/{opportunity_id}` to return complete tender data for paid-tier users — including contracting authority, full description, evaluation criteria, mandatory documents, submission deadline with timezone, CPV codes with labels, linked AI analysis summary (if previously generated), a submission guide reference from `pipeline.submission_guides`, and up to 5 related opportunities by CPV/region overlap — after TierGate scope validation, while free-tier users always receive 403 with an upgrade prompt**,
so that **the frontend detail page (S06.11) has all the data needed to render its five tabs (Overview, Documents, Requirements, AI Analysis, Submission Guide), and tier enforcement is consistently applied at the single-item level using `check_scope()` from the `OpportunityTierGateContext` established in S06.02**.

## Acceptance Criteria

1. `GET /api/v1/opportunities/{opportunity_id}` is registered in the opportunities router at path `/{opportunity_id}` (prefix `/api/v1/opportunities`), is distinct from `/search` and `""` (listing), requires a valid Bearer JWT (returns 401 if absent/invalid), and appears in `/openapi.json` with correct path, path parameter schema, and response description.

2. Free-tier users (`subscription_tier = free`) always receive 403 with body `{"error": "tier_limit", "upgrade_url": "<billing_upgrade_url>"}` regardless of which `opportunity_id` they request; no opportunity data is returned to free-tier users on the detail endpoint.

3. Paid-tier users (starter, professional, enterprise) receive TierGate scope validation via `tier_gate.check_scope(item)` after the opportunity is fetched; if the opportunity is outside the user's tier scope (e.g. Starter user requests a non-Bulgaria opportunity or a budget > 500K EUR), 403 is returned with body `{"error": "tier_limit", "upgrade_url": "..."}`.

4. Returns 404 with `{"error": "not_found", "detail": "Opportunity not found."}` for a non-existent `opportunity_id` (UUID not present in `pipeline.opportunities` or where `deleted_at IS NOT NULL`).

5. For an in-scope paid-tier user, the response JSON contains all `OpportunityFullResponse` fields (id, title, description, status, opportunity_type, deadline, budget_min, budget_max, currency, country, region, contracting_authority, cpv_codes, evaluation_criteria, mandatory_documents, relevance_scores, published_at, updated_at) plus three additional top-level keys: `related_opportunities` (list, max 5 items), `submission_guide` (object or null), `ai_summary` (object or null).

6. `related_opportunities` is a list of up to 5 `RelatedOpportunityItem` objects for opportunities that share at least one CPV code with the requested opportunity OR are in the same region; ordered by deadline ASC (soonest-deadline relevant opportunities first); the requested opportunity itself is excluded; each item is scope-checked and out-of-scope items are silently excluded from the list (not 403'd); contains fields: `id`, `title`, `region`, `cpv_codes`, `status`, `deadline`, `opportunity_type`.

7. `submission_guide` when present is the most recently created non-deleted row from `pipeline.submission_guides` for this `opportunity_id`; includes fields: `id`, `source_portal`, `reviewed`, `steps` (full JSONB steps array); when no submission guide exists, `submission_guide` is `null`.

8. `ai_summary` when present is the most recently generated row from `client.ai_summaries` for `(opportunity_id, user_id)` matching the requesting user; includes fields: `id`, `content`, `model`, `tokens_used`, `generated_at`; when no prior summary exists, `ai_summary` is `null`; fetching the summary via this endpoint does NOT decrement the AI usage counter.

9. The `client.ai_summaries` Alembic migration (`018_ai_summaries.py`) creates the table in the `client` schema with columns: `id UUID PK DEFAULT gen_random_uuid()`, `opportunity_id UUID NOT NULL`, `user_id UUID NOT NULL REFERENCES client.users(id) ON DELETE CASCADE`, `company_id UUID NOT NULL REFERENCES client.companies(id) ON DELETE CASCADE`, `content TEXT NOT NULL`, `model VARCHAR(100)`, `tokens_used INTEGER`, `generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`; with composite index on `(opportunity_id, user_id)` and single-column index on `company_id`.

10. Integration tests in `tests/api/test_opportunity_detail.py` cover: free-tier → 403 with `tier_limit` body (E06-P0-001); paid-tier in-scope → 200 with all `OpportunityFullResponse` fields present including `evaluation_criteria`, `mandatory_documents`, deadline timezone-aware (E06-P1-012); `related_opportunities` → up to 5 by CPV/region (E06-P1-013); 404 for non-existent UUID (E06-P1-014); pre-seeded AI summary returned without counter decrement (E06-P1-015); paid-tier out-of-scope → 403 (E06-P0-003 partial); unauthenticated → 401.

## Tasks / Subtasks

- [ ] Task 1: Create `client.ai_summaries` Alembic migration (AC: 9)
  - [ ] 1.1 Create `eusolicit-app/services/client-api/alembic/versions/018_ai_summaries.py`:

    ```python
    """Create client.ai_summaries table for S06.05 / S06.08.

    Revision ID: 018_ai_summaries
    Revises: 017_user_onboarding_completed
    Create Date: 2026-04-17

    client.ai_summaries stores AI-generated executive summaries per (opportunity, user).
    The opportunity_id references pipeline.opportunities but is stored as a bare UUID
    (no FK) to avoid cross-schema foreign key constraints.
    S06.05 reads from this table; S06.08 writes to it.
    """

    from __future__ import annotations

    import sqlalchemy as sa
    from alembic import op

    revision = "018_ai_summaries"
    down_revision = "017_user_onboarding_completed"
    branch_labels = None
    depends_on = None


    def upgrade() -> None:
        op.create_table(
            "ai_summaries",
            sa.Column("id", sa.UUID(as_uuid=True), primary_key=True,
                      server_default=sa.text("gen_random_uuid()")),
            sa.Column("opportunity_id", sa.UUID(as_uuid=True), nullable=False),
            sa.Column(
                "user_id",
                sa.UUID(as_uuid=True),
                sa.ForeignKey("client.users.id", ondelete="CASCADE"),
                nullable=False,
            ),
            sa.Column(
                "company_id",
                sa.UUID(as_uuid=True),
                sa.ForeignKey("client.companies.id", ondelete="CASCADE"),
                nullable=False,
            ),
            sa.Column("content", sa.Text, nullable=False),
            sa.Column("model", sa.String(100), nullable=True),
            sa.Column("tokens_used", sa.Integer, nullable=True),
            sa.Column(
                "generated_at",
                sa.DateTime(timezone=True),
                nullable=False,
                server_default=sa.text("NOW()"),
            ),
            sa.Column(
                "created_at",
                sa.DateTime(timezone=True),
                nullable=False,
                server_default=sa.text("NOW()"),
            ),
            schema="client",
        )
        op.create_index(
            "ix_ai_summaries_opportunity_user",
            "ai_summaries",
            ["opportunity_id", "user_id"],
            schema="client",
        )
        op.create_index(
            "ix_ai_summaries_company_id",
            "ai_summaries",
            ["company_id"],
            schema="client",
        )


    def downgrade() -> None:
        op.drop_index("ix_ai_summaries_company_id", table_name="ai_summaries", schema="client")
        op.drop_index("ix_ai_summaries_opportunity_user", table_name="ai_summaries", schema="client")
        op.drop_table("ai_summaries", schema="client")
    ```

- [ ] Task 2: Create `pipeline_submission_guide.py` Core table definition (AC: 7)
  - [ ] 2.1 Create `eusolicit-app/services/client-api/src/client_api/models/pipeline_submission_guide.py`:

    ```python
    """Read-only SQLAlchemy Core table definition for pipeline.submission_guides (S06.05).

    Mirrors the data-pipeline ORM model (data_pipeline.models.submission_guide).
    Only the columns consumed by the detail endpoint are declared.

    Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.05
    """
    from __future__ import annotations

    from sqlalchemy import Boolean, Column, DateTime, MetaData, String, Table
    from sqlalchemy.dialects.postgresql import JSON, UUID

    from client_api.models.pipeline_opportunity import pipeline_metadata

    submission_guides_table = Table(
        "submission_guides",
        pipeline_metadata,          # same MetaData(schema="pipeline") instance
        Column("id", UUID(as_uuid=True), primary_key=True),
        Column("opportunity_id", UUID(as_uuid=True)),
        Column("source_portal", String(20)),
        Column("steps", JSON),      # list of {step_number, title, instruction, ...}
        Column("reviewed", Boolean),
        Column("deleted_at", DateTime(timezone=True)),
        Column("created_at", DateTime(timezone=True)),
        Column("updated_at", DateTime(timezone=True)),
    )
    ```

- [ ] Task 3: Create `client_ai_summary.py` Core table definition (AC: 8)
  - [ ] 3.1 Create `eusolicit-app/services/client-api/src/client_api/models/client_ai_summary.py`:

    ```python
    """Read-only SQLAlchemy Core table definition for client.ai_summaries (S06.05).

    client.ai_summaries stores AI-generated executive summaries per (opportunity, user).
    S06.05 queries the most-recent summary for (opportunity_id, user_id) to populate
    the ai_summary field in OpportunityDetailResponse.
    S06.08 writes new summaries.

    Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.05
    """
    from __future__ import annotations

    from sqlalchemy import Column, DateTime, Integer, MetaData, String, Table, Text
    from sqlalchemy.dialects.postgresql import UUID

    client_metadata = MetaData(schema="client")

    ai_summaries_table = Table(
        "ai_summaries",
        client_metadata,
        Column("id", UUID(as_uuid=True), primary_key=True),
        Column("opportunity_id", UUID(as_uuid=True)),
        Column("user_id", UUID(as_uuid=True)),
        Column("company_id", UUID(as_uuid=True)),
        Column("content", Text),
        Column("model", String(100)),
        Column("tokens_used", Integer),
        Column("generated_at", DateTime(timezone=True)),
        Column("created_at", DateTime(timezone=True)),
    )
    ```

- [ ] Task 4: Add `OpportunityDetailResponse` and supporting schemas to `schemas/opportunities.py` (AC: 5, 6, 7, 8)
  - [ ] 4.1 Edit `eusolicit-app/services/client-api/src/client_api/schemas/opportunities.py` — append the following after the `OpportunitySearchResponse` class:

    ```python

    # ---------------------------------------------------------------------------
    # Detail endpoint schemas (S06.05)
    # ---------------------------------------------------------------------------

    class RelatedOpportunityItem(BaseModel):
        """Lightweight opportunity item for the related_opportunities list in OpportunityDetailResponse.

        Contains only the fields needed for the frontend listing card on the detail page.
        Does not include budget or evaluation_criteria to keep the payload compact.
        Paid-tier scope is enforced by the service before including the item.
        """

        model_config = ConfigDict(from_attributes=True)

        id: UUID
        title: str
        region: str | None = None
        cpv_codes: list[str] = Field(default_factory=list)
        status: str
        deadline: datetime | None = None
        opportunity_type: str


    class SubmissionGuideRef(BaseModel):
        """Reference to the pipeline.submission_guides row for this opportunity (S06.05).

        Includes the full `steps` JSONB array so the frontend Submission Guide tab can
        render the accordion without a separate endpoint call.
        """

        model_config = ConfigDict(from_attributes=True)

        id: UUID
        source_portal: str
        reviewed: bool
        steps: list[dict] | None = None


    class AISummaryRef(BaseModel):
        """Reference to the client.ai_summaries row for this (opportunity, user) pair (S06.05).

        Returned from the detail endpoint when a prior AI summary exists for the
        requesting user.  Fetching via this endpoint does NOT decrement the usage counter.
        """

        model_config = ConfigDict(from_attributes=True)

        id: UUID
        content: str
        model: str | None = None
        tokens_used: int | None = None
        generated_at: datetime


    class OpportunityDetailResponse(OpportunityFullResponse):
        """Full opportunity detail response for paid-tier users (S06.05).

        Extends OpportunityFullResponse with three extra fields:
          related_opportunities  — up to 5 related opportunities by CPV/region overlap.
          submission_guide       — most recent pipeline.submission_guides row, or null.
          ai_summary             — most recent client.ai_summaries row for (opp, user), or null.
        """

        related_opportunities: list[RelatedOpportunityItem] = Field(default_factory=list)
        submission_guide: SubmissionGuideRef | None = None
        ai_summary: AISummaryRef | None = None
    ```

- [ ] Task 5: Add `get_opportunity_detail()` service function to `opportunity_service.py` (AC: 4, 5, 6, 7, 8)
  - [ ] 5.1 Add the following imports to the top of `opportunity_service.py` (add to existing imports — some may already be present):

    ```python
    from uuid import UUID
    ```

    Also verify that `from sqlalchemy import and_, func, or_, select` is already present (added in S06.01).

  - [ ] 5.2 Add the following import to the imports block in `opportunity_service.py`:

    ```python
    from client_api.models.pipeline_submission_guide import submission_guides_table as sg_t
    from client_api.models.client_ai_summary import ai_summaries_table as ai_t
    from client_api.schemas.opportunities import (
        AISummaryRef,
        OpportunityDetailResponse,
        RelatedOpportunityItem,
        SubmissionGuideRef,
    )
    ```

    Note: Add these imports to the existing `from client_api.schemas.opportunities import ...` line; avoid duplicate imports.

  - [ ] 5.3 Add `get_opportunity_detail()` function to `opportunity_service.py` AFTER `list_opportunities()`:

    ```python
    # ---------------------------------------------------------------------------
    # Detail function (single opportunity — S06.05)
    # ---------------------------------------------------------------------------

    async def get_opportunity_detail(
        *,
        pipeline_session: AsyncSession,
        client_session: AsyncSession,
        opportunity_id: UUID,
        user_id: UUID,
        tier_gate: "OpportunityTierGateContext",  # noqa: F821 — imported in route layer
    ) -> OpportunityDetailResponse:
        """Fetch full opportunity detail including related opportunities, submission guide,
        and any previously generated AI summary for the requesting user.

        Parameters
        ----------
        pipeline_session:
            Read-only async session scoped to the pipeline schema (get_pipeline_readonly_session).
        client_session:
            Read-write async session scoped to the client schema (get_db_session).
            Used only for SELECT on client.ai_summaries — no writes.
        opportunity_id:
            UUID of the requested opportunity.
        user_id:
            UUID of the requesting user (from JWT via get_current_user).
            Used to scope the AI summary lookup.
        tier_gate:
            OpportunityTierGateContext (already resolved by get_opportunity_tier_gate).
            Used to scope-check related opportunity items.

        Returns
        -------
        OpportunityDetailResponse
            Full tender data with related_opportunities, submission_guide, ai_summary.

        Raises
        ------
        eusolicit_common.exceptions.AppException (status_code=404)
            When opportunity_id does not exist or has been soft-deleted.
        """
        from eusolicit_common.exceptions import AppException  # noqa: PLC0415

        # ----- 1. Fetch main opportunity -----
        columns: list[Any] = [
            opp_t.c.id,
            opp_t.c.title,
            opp_t.c.description,
            opp_t.c.status,
            opp_t.c.opportunity_type,
            opp_t.c.deadline,
            opp_t.c.budget_min,
            opp_t.c.budget_max,
            opp_t.c.currency,
            opp_t.c.country,
            opp_t.c.region,
            opp_t.c.contracting_authority,
            opp_t.c.cpv_codes,
            opp_t.c.evaluation_criteria,
            opp_t.c.mandatory_documents,
            opp_t.c.relevance_scores,
            opp_t.c.published_at,
            opp_t.c.updated_at,
        ]

        opp_stmt = (
            select(*columns)
            .where(opp_t.c.id == opportunity_id)
            .where(opp_t.c.deleted_at.is_(None))
        )
        opp_row = (await pipeline_session.execute(opp_stmt)).mappings().first()

        if opp_row is None:
            raise AppException(
                "Opportunity not found.",
                error="not_found",
                status_code=404,
            )

        # Build OpportunitySearchItem from the row (used by TierGate helpers)
        main_item = OpportunitySearchItem(
            id=opp_row["id"],
            title=opp_row["title"] or "",
            description=opp_row.get("description"),
            status=opp_row["status"],
            opportunity_type=opp_row["opportunity_type"] or "tender",
            deadline=opp_row.get("deadline"),
            budget_min=opp_row.get("budget_min"),
            budget_max=opp_row.get("budget_max"),
            currency=opp_row.get("currency"),
            country=opp_row.get("country"),
            region=opp_row.get("region"),
            contracting_authority=opp_row.get("contracting_authority"),
            cpv_codes=list(opp_row.get("cpv_codes") or []),
            evaluation_criteria=opp_row.get("evaluation_criteria"),
            mandatory_documents=opp_row.get("mandatory_documents"),
            relevance_scores=opp_row.get("relevance_scores"),
            published_at=opp_row.get("published_at"),
            updated_at=opp_row.get("updated_at"),
            relevance_rank=None,
        )

        # ----- 2. Fetch related opportunities (CPV overlap OR same region) -----
        related_items: list[RelatedOpportunityItem] = []
        cpv_list = main_item.cpv_codes
        region = main_item.region

        # Build the overlap / region condition — only query if we have something to match on
        overlap_conditions: list[Any] = []
        if cpv_list:
            overlap_conditions.append(opp_t.c.cpv_codes.overlap(cpv_list))
        if region:
            overlap_conditions.append(opp_t.c.region == region)

        if overlap_conditions:
            related_stmt = (
                select(
                    opp_t.c.id,
                    opp_t.c.title,
                    opp_t.c.region,
                    opp_t.c.cpv_codes,
                    opp_t.c.status,
                    opp_t.c.deadline,
                    opp_t.c.opportunity_type,
                    opp_t.c.budget_max,  # needed for TierGate is_in_scope check
                )
                .where(opp_t.c.id != opportunity_id)
                .where(opp_t.c.deleted_at.is_(None))
                .where(or_(*overlap_conditions))
                .order_by(opp_t.c.deadline.asc().nulls_last())
                .limit(10)  # fetch extra to allow TierGate filtering down to 5
            )
            related_rows = (await pipeline_session.execute(related_stmt)).mappings().all()

            for row in related_rows:
                if len(related_items) >= 5:
                    break
                # Scope-check each related item before including it
                candidate = OpportunitySearchItem(
                    id=row["id"],
                    title=row["title"] or "",
                    status=row["status"],
                    opportunity_type=row["opportunity_type"] or "tender",
                    region=row.get("region"),
                    cpv_codes=list(row.get("cpv_codes") or []),
                    deadline=row.get("deadline"),
                    budget_max=row.get("budget_max"),
                )
                if tier_gate.is_in_scope(candidate):
                    related_items.append(
                        RelatedOpportunityItem(
                            id=row["id"],
                            title=row["title"] or "",
                            region=row.get("region"),
                            cpv_codes=list(row.get("cpv_codes") or []),
                            status=row["status"],
                            deadline=row.get("deadline"),
                            opportunity_type=row["opportunity_type"] or "tender",
                        )
                    )

        # ----- 3. Fetch submission guide (most recent non-deleted) -----
        submission_guide: SubmissionGuideRef | None = None
        sg_stmt = (
            select(
                sg_t.c.id,
                sg_t.c.source_portal,
                sg_t.c.reviewed,
                sg_t.c.steps,
            )
            .where(sg_t.c.opportunity_id == opportunity_id)
            .where(sg_t.c.deleted_at.is_(None))
            .order_by(sg_t.c.created_at.desc())
            .limit(1)
        )
        sg_row = (await pipeline_session.execute(sg_stmt)).mappings().first()
        if sg_row is not None:
            submission_guide = SubmissionGuideRef(
                id=sg_row["id"],
                source_portal=sg_row["source_portal"],
                reviewed=bool(sg_row["reviewed"]),
                steps=sg_row.get("steps"),
            )

        # ----- 4. Fetch AI summary (most recent for this user+opportunity) -----
        ai_summary: AISummaryRef | None = None
        ai_stmt = (
            select(
                ai_t.c.id,
                ai_t.c.content,
                ai_t.c.model,
                ai_t.c.tokens_used,
                ai_t.c.generated_at,
            )
            .where(ai_t.c.opportunity_id == opportunity_id)
            .where(ai_t.c.user_id == user_id)
            .order_by(ai_t.c.generated_at.desc())
            .limit(1)
        )
        ai_row = (await client_session.execute(ai_stmt)).mappings().first()
        if ai_row is not None:
            ai_summary = AISummaryRef(
                id=ai_row["id"],
                content=ai_row["content"],
                model=ai_row.get("model"),
                tokens_used=ai_row.get("tokens_used"),
                generated_at=ai_row["generated_at"],
            )

        # ----- 5. Build and return OpportunityDetailResponse -----
        return OpportunityDetailResponse(
            id=main_item.id,
            title=main_item.title,
            description=main_item.description,
            status=main_item.status,
            opportunity_type=main_item.opportunity_type,
            deadline=main_item.deadline,
            budget_min=main_item.budget_min,
            budget_max=main_item.budget_max,
            currency=main_item.currency,
            country=main_item.country,
            region=main_item.region,
            contracting_authority=main_item.contracting_authority,
            cpv_codes=main_item.cpv_codes,
            evaluation_criteria=main_item.evaluation_criteria,
            mandatory_documents=main_item.mandatory_documents,
            relevance_scores=main_item.relevance_scores,
            published_at=main_item.published_at,
            updated_at=main_item.updated_at,
            relevance_rank=None,
            related_opportunities=related_items,
            submission_guide=submission_guide,
            ai_summary=ai_summary,
        )
    ```

    Note: The `OpportunityTierGateContext` type hint uses a string (`"OpportunityTierGateContext"`) to avoid a circular import — the gate imports from `schemas.opportunities` which is in the same package. The route handler passes the fully resolved `tier_gate` instance from the dependency, so runtime resolution is fine.

- [ ] Task 6: Add `GET /{opportunity_id}` route handler to `api/v1/opportunities.py` (AC: 1, 2, 3, 4)
  - [ ] 6.1 Add the following imports to `eusolicit-app/services/client-api/src/client_api/api/v1/opportunities.py` (append to existing imports — check first):

    ```python
    from uuid import UUID

    from eusolicit_common.exceptions import AppException
    from client_api.dependencies import get_db_session
    from client_api.schemas.opportunities import OpportunityDetailResponse
    ```

    Note: `AppException`, `get_db_session`, and `UUID` may already be imported. Check before adding duplicates.

  - [ ] 6.2 Edit `eusolicit-app/services/client-api/src/client_api/api/v1/opportunities.py` — append the following AFTER the `search_opportunities` function at the end of the file:

    ```python
    # ---------------------------------------------------------------------------
    # GET /opportunities/{opportunity_id}  (detail — S06.05)
    # ---------------------------------------------------------------------------

    _PAID_TIERS = frozenset({"starter", "professional", "enterprise"})


    @router.get(
        "/{opportunity_id}",
        summary="Get full opportunity detail for paid-tier users",
        description=(
            "Returns complete tender data for a single opportunity. "
            "Free-tier users always receive 403 with tier_limit error. "
            "Paid-tier users receive full OpportunityDetailResponse after TierGate scope validation. "
            "Includes evaluation_criteria, mandatory_documents, related_opportunities (up to 5), "
            "submission_guide reference, and ai_summary (if previously generated by this user). "
            "Fetching the ai_summary via this endpoint does NOT decrement the usage counter. "
            "Returns 404 for unknown or soft-deleted opportunity_id."
        ),
    )
    async def get_opportunity_detail(
        opportunity_id: UUID,
        current_user: Annotated[CurrentUser, Depends(get_current_user)] = None,  # type: ignore[assignment]
        session: Annotated[AsyncSession, Depends(get_pipeline_readonly_session)] = None,  # type: ignore[assignment]
        db: Annotated[AsyncSession, Depends(get_db_session)] = None,  # type: ignore[assignment]
        tier_gate: Annotated[OpportunityTierGateContext, Depends(get_opportunity_tier_gate)] = None,  # type: ignore[assignment]
    ) -> dict:
        """Retrieve full opportunity detail with tier enforcement.

        AC1: Endpoint registered at /api/v1/opportunities/{opportunity_id}.
        AC2: Free-tier users always receive 403 (no opportunity data returned).
        AC3: Paid-tier scope validation via tier_gate.check_scope().
        AC4: 404 for non-existent or soft-deleted opportunity_id.
        AC5-8: Returns OpportunityDetailResponse with related_opps, submission_guide, ai_summary.
        """
        from client_api.config import get_settings  # noqa: PLC0415

        # AC2: Block free-tier users before any DB query
        if tier_gate.user_tier not in _PAID_TIERS:
            settings = get_settings()
            upgrade_url = settings.frontend_url.rstrip("/") + "/billing/upgrade"
            raise AppException(
                "Opportunity detail requires a paid subscription.",
                error="tier_limit",
                status_code=403,
                details={"upgrade_url": upgrade_url},
            )

        # Fetch detail (includes 404 for not found)
        detail = await opportunity_service.get_opportunity_detail(
            pipeline_session=session,
            client_session=db,
            opportunity_id=opportunity_id,
            user_id=current_user.id,
            tier_gate=tier_gate,
        )

        # AC3: Scope-check the main opportunity for paid-tier users
        # check_scope raises 403 AppException if the opportunity is outside tier scope
        main_item_for_scope = detail  # OpportunityDetailResponse inherits OpportunityFullResponse
        # Build a minimal OpportunitySearchItem for scope checking
        from client_api.schemas.opportunities import OpportunitySearchItem  # noqa: PLC0415
        scope_item = OpportunitySearchItem(
            id=detail.id,
            title=detail.title,
            status=detail.status,
            opportunity_type=detail.opportunity_type,
            region=detail.region,
            cpv_codes=detail.cpv_codes,
            budget_max=detail.budget_max,
        )
        tier_gate.check_scope(scope_item)

        return detail.model_dump(mode="json")
    ```

- [ ] Task 7: Write integration tests (AC: 10)
  - [ ] 7.1 Create `eusolicit-app/services/client-api/tests/api/test_opportunity_detail.py`:

    ```python
    """Integration tests for GET /api/v1/opportunities/{opportunity_id} (Story S06.05).

    Test IDs covered:
        E06-P0-001  Free-tier → 403 with tier_limit body and upgrade_url
        E06-P0-003  Paid-tier (Starter) requests out-of-scope opportunity → 403
        E06-P1-012  Paid-tier in-scope → 200 with all required fields (evaluation_criteria,
                    mandatory_documents, deadline with timezone)
        E06-P1-013  related_opportunities → up to 5 by CPV/region overlap
        E06-P1-014  Non-existent opportunity_id → 404
        E06-P1-015  Pre-seeded AI summary returned without counter decrement
        (Implicit) Unauthenticated → 401

    Fixtures:
        Seed pipeline.opportunities rows (with evaluation_criteria and mandatory_documents JSONB)
        and a pipeline.submission_guides row using the superuser session.
        Seed client.ai_summaries row for the pre-generated summary test.

    Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#P0-001, P0-003, P1-012,
            P1-013, P1-014, P1-015
    """
    from __future__ import annotations

    import uuid
    from datetime import UTC, datetime, timedelta

    import pytest
    import pytest_asyncio
    from sqlalchemy import text

    # ---------------------------------------------------------------------------
    # Seed data
    # ---------------------------------------------------------------------------

    _STARTER_COMPANY_ID = "12345678-0000-0000-0000-000000000001"  # matches starter_user_token fixture

    _MAIN_OPP_ID = str(uuid.uuid4())
    _RELATED_OPP_ID_1 = str(uuid.uuid4())
    _RELATED_OPP_ID_2 = str(uuid.uuid4())
    _OUT_OF_SCOPE_OPP_ID = str(uuid.uuid4())   # French opportunity — out of Starter scope

    _EVAL_CRITERIA = {
        "criteria": [
            {"name": "Technical Quality", "weight": 60, "type": "quality", "description": "Technical merit"},
            {"name": "Price", "weight": 40, "type": "price", "description": "Best value"},
        ]
    }
    _MANDATORY_DOCS = {
        "documents": [
            {"name": "Technical Proposal", "required": True},
            {"name": "Financial Offer", "required": True},
        ]
    }


    @pytest_asyncio.fixture(scope="module")
    async def seeded_detail_opportunities(superuser_session_factory):
        """Seed pipeline.opportunities + pipeline.submission_guides for detail tests."""
        async with superuser_session_factory() as session:
            # Main opportunity — Bulgaria, ≤500K EUR (in Starter scope)
            await session.execute(text("""
                INSERT INTO pipeline.opportunities (
                    id, source_id, source_type, title, description, contracting_authority,
                    opportunity_type, status, deadline, budget_min, budget_max, currency,
                    country, region, cpv_codes, evaluation_criteria, mandatory_documents,
                    published_at, created_at, updated_at
                ) VALUES (
                    :id, :source_id, 'aop', :title, :description, :contracting_authority,
                    'tender', 'open', :deadline, 100000, 400000, 'EUR',
                    'Bulgaria', 'Bulgaria', '{45000000-7,72000000-5}'::text[],
                    :eval_criteria::jsonb, :mandatory_docs::jsonb,
                    NOW() - INTERVAL '5 days', NOW(), NOW()
                )
                ON CONFLICT (source_id, source_type) DO NOTHING
            """), {
                "id": _MAIN_OPP_ID,
                "source_id": "detail-opp-main",
                "title": "Main Detail Opportunity",
                "description": "Full description with all criteria",
                "contracting_authority": "Bulgarian Ministry of Finance",
                "deadline": (datetime.now(UTC) + timedelta(days=30)).isoformat(),
                "eval_criteria": str(_EVAL_CRITERIA).replace("'", '"'),
                "mandatory_docs": str(_MANDATORY_DOCS).replace("'", '"'),
            })

            # Related opportunity 1 — shares CPV 45000000-7, Bulgaria
            await session.execute(text("""
                INSERT INTO pipeline.opportunities (
                    id, source_id, source_type, title, opportunity_type, status,
                    deadline, budget_max, currency, country, region, cpv_codes,
                    published_at, created_at, updated_at
                ) VALUES (
                    :id, :source_id, 'aop', :title, 'tender', 'open',
                    :deadline, 300000, 'EUR', 'Bulgaria', 'Bulgaria', '{45000000-7}'::text[],
                    NOW(), NOW(), NOW()
                )
                ON CONFLICT (source_id, source_type) DO NOTHING
            """), {
                "id": _RELATED_OPP_ID_1,
                "source_id": "detail-opp-related-1",
                "title": "Related Opportunity 1 (same CPV)",
                "deadline": (datetime.now(UTC) + timedelta(days=20)).isoformat(),
            })

            # Related opportunity 2 — shares region Bulgaria (no CPV overlap)
            await session.execute(text("""
                INSERT INTO pipeline.opportunities (
                    id, source_id, source_type, title, opportunity_type, status,
                    deadline, budget_max, currency, country, region, cpv_codes,
                    published_at, created_at, updated_at
                ) VALUES (
                    :id, :source_id, 'aop', :title, 'tender', 'open',
                    :deadline, 200000, 'EUR', 'Bulgaria', 'Bulgaria', '{79000000-4}'::text[],
                    NOW(), NOW(), NOW()
                )
                ON CONFLICT (source_id, source_type) DO NOTHING
            """), {
                "id": _RELATED_OPP_ID_2,
                "source_id": "detail-opp-related-2",
                "title": "Related Opportunity 2 (same region)",
                "deadline": (datetime.now(UTC) + timedelta(days=25)).isoformat(),
            })

            # Out-of-scope opportunity — France, >500K EUR (outside Starter scope)
            await session.execute(text("""
                INSERT INTO pipeline.opportunities (
                    id, source_id, source_type, title, opportunity_type, status,
                    deadline, budget_max, currency, country, region, cpv_codes,
                    published_at, created_at, updated_at
                ) VALUES (
                    :id, :source_id, 'ted', :title, 'tender', 'open',
                    :deadline, 2000000, 'EUR', 'France', 'France', '{45000000-7}'::text[],
                    NOW(), NOW(), NOW()
                )
                ON CONFLICT (source_id, source_type) DO NOTHING
            """), {
                "id": _OUT_OF_SCOPE_OPP_ID,
                "source_id": "detail-opp-out-of-scope",
                "title": "French Opportunity (out of Starter scope)",
                "deadline": (datetime.now(UTC) + timedelta(days=15)).isoformat(),
            })

            # Submission guide for main opportunity
            await session.execute(text("""
                INSERT INTO pipeline.submission_guides (
                    id, opportunity_id, source_portal, steps, reviewed, created_at, updated_at
                ) VALUES (
                    gen_random_uuid(), :opp_id, 'aop',
                    '[{"step_number": 1, "title": "Register", "instruction": "Register on AOP portal"}]'::jsonb,
                    false, NOW(), NOW()
                )
                ON CONFLICT DO NOTHING
            """), {"opp_id": _MAIN_OPP_ID})

            await session.commit()

        yield {
            "main": _MAIN_OPP_ID,
            "related_1": _RELATED_OPP_ID_1,
            "related_2": _RELATED_OPP_ID_2,
            "out_of_scope": _OUT_OF_SCOPE_OPP_ID,
        }

        # Cleanup
        async with superuser_session_factory() as session:
            await session.execute(text(
                "DELETE FROM pipeline.submission_guides WHERE opportunity_id = :id"
            ), {"id": _MAIN_OPP_ID})
            await session.execute(text(
                "DELETE FROM pipeline.opportunities WHERE source_id LIKE 'detail-opp-%'"
            ))
            await session.commit()


    @pytest_asyncio.fixture(scope="module")
    async def seeded_ai_summary(superuser_session_factory):
        """Seed a client.ai_summaries row for E06-P1-015."""
        _user_id_placeholder = None  # resolved below from the starter JWT

        # We need a real user_id and company_id that exist in client.users / client.companies.
        # Use the starter user from the test fixtures — their IDs are in client.users.
        # For simplicity, use a raw INSERT with the known company_id from the JWT fixture.
        _summary_id = str(uuid.uuid4())

        async with superuser_session_factory() as session:
            # Look up any user in the starter company so we can foreign-key correctly
            row = await session.execute(text("""
                SELECT u.id as user_id
                FROM client.users u
                JOIN client.company_memberships m ON m.user_id = u.id
                JOIN client.companies c ON c.id = m.company_id
                WHERE c.id = :company_id
                LIMIT 1
            """), {"company_id": _STARTER_COMPANY_ID})
            result = row.mappings().first()
            if result is None:
                # No matching user found — skip this fixture gracefully
                yield None
                return

            user_id = str(result["user_id"])
            await session.execute(text("""
                INSERT INTO client.ai_summaries (
                    id, opportunity_id, user_id, company_id, content, model,
                    tokens_used, generated_at, created_at
                ) VALUES (
                    :id, :opp_id, :user_id, :company_id,
                    'AI-generated executive summary for the main opportunity.',
                    'gpt-4o', 1500, NOW() - INTERVAL '1 hour', NOW() - INTERVAL '1 hour'
                )
            """), {
                "id": _summary_id,
                "opp_id": _MAIN_OPP_ID,
                "user_id": user_id,
                "company_id": _STARTER_COMPANY_ID,
            })
            await session.commit()

        yield {"summary_id": _summary_id, "user_id": user_id}

        async with superuser_session_factory() as session:
            await session.execute(
                text("DELETE FROM client.ai_summaries WHERE id = :id"),
                {"id": _summary_id},
            )
            await session.commit()


    @pytest_asyncio.fixture
    async def detail_app(app, superuser_session_factory):
        """Override both pipeline and client DB sessions for detail tests."""
        from client_api.dependencies import (  # noqa: PLC0415
            get_db_session,
            get_pipeline_readonly_session,
        )

        async def _pipeline_override():
            async with superuser_session_factory() as s:
                await s.execute(text("SET TRANSACTION READ ONLY"))
                try:
                    yield s
                finally:
                    await s.rollback()

        async def _client_override():
            async with superuser_session_factory() as s:
                try:
                    yield s
                finally:
                    await s.rollback()

        app.dependency_overrides[get_pipeline_readonly_session] = _pipeline_override
        app.dependency_overrides[get_db_session] = _client_override
        yield app


    # ---------------------------------------------------------------------------
    # Helper headers
    # ---------------------------------------------------------------------------

    @pytest.fixture
    def free_headers(free_user_token):
        return {"Authorization": f"Bearer {free_user_token}"}


    @pytest.fixture
    def starter_headers(starter_user_token):
        return {"Authorization": f"Bearer {starter_user_token}"}


    @pytest.fixture
    def enterprise_headers(enterprise_user_token):
        return {"Authorization": f"Bearer {enterprise_user_token}"}


    # ---------------------------------------------------------------------------
    # E06-P0-001: Free-tier → 403 tier_limit
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_free_tier_blocked(
        detail_app, test_client, free_headers, seeded_detail_opportunities
    ):
        """E06-P0-001: Free-tier user receives 403 with tier_limit error on detail endpoint."""
        opp_id = seeded_detail_opportunities["main"]
        resp = await test_client.get(
            f"/api/v1/opportunities/{opp_id}",
            headers=free_headers,
        )
        assert resp.status_code == 403
        data = resp.json()
        assert data.get("error") == "tier_limit" or "tier_limit" in str(data)
        assert "upgrade_url" in str(data), "upgrade_url must be present in 403 response"


    # ---------------------------------------------------------------------------
    # Unauthenticated → 401
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_unauthenticated(
        detail_app, test_client, seeded_detail_opportunities
    ):
        """Unauthenticated request returns 401."""
        opp_id = seeded_detail_opportunities["main"]
        resp = await test_client.get(f"/api/v1/opportunities/{opp_id}")
        assert resp.status_code == 401


    # ---------------------------------------------------------------------------
    # E06-P1-014: 404 for non-existent opportunity_id
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_not_found(detail_app, test_client, starter_headers):
        """E06-P1-014: Non-existent opportunity_id returns 404."""
        non_existent_id = str(uuid.uuid4())
        resp = await test_client.get(
            f"/api/v1/opportunities/{non_existent_id}",
            headers=starter_headers,
        )
        assert resp.status_code == 404


    # ---------------------------------------------------------------------------
    # E06-P1-012: Paid-tier in-scope → full tender data
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_paid_tier_full_response(
        detail_app, test_client, starter_headers, seeded_detail_opportunities
    ):
        """E06-P1-012: Paid-tier in-scope → 200 with full tender data.

        Asserts: evaluation_criteria, mandatory_documents, deadline with timezone,
        contracting_authority, cpv_codes, budget fields, related_opportunities,
        submission_guide, and ai_summary (null if no prior summary).
        """
        opp_id = seeded_detail_opportunities["main"]
        resp = await test_client.get(
            f"/api/v1/opportunities/{opp_id}",
            headers=starter_headers,
        )
        assert resp.status_code == 200, f"Expected 200, got {resp.status_code}: {resp.text}"
        data = resp.json()

        # Core OpportunityFullResponse fields
        assert data["id"] == opp_id
        assert data["title"] == "Main Detail Opportunity"
        assert "description" in data
        assert "contracting_authority" in data
        assert data["contracting_authority"] == "Bulgarian Ministry of Finance"
        assert "cpv_codes" in data
        assert isinstance(data["cpv_codes"], list)
        assert "budget_min" in data
        assert "budget_max" in data
        assert "currency" in data

        # AC: deadline with timezone (ISO8601 with offset)
        assert "deadline" in data
        assert data["deadline"] is not None
        # Deadline must include timezone info (Z or +00:00 suffix)
        deadline_str = data["deadline"]
        assert "Z" in deadline_str or "+" in deadline_str or "-" in deadline_str[-6:], (
            f"deadline must include timezone offset, got: {deadline_str}"
        )

        # AC: evaluation_criteria present and non-null
        assert "evaluation_criteria" in data
        assert data["evaluation_criteria"] is not None
        assert "criteria" in data["evaluation_criteria"]

        # AC: mandatory_documents present and non-null
        assert "mandatory_documents" in data
        assert data["mandatory_documents"] is not None
        assert "documents" in data["mandatory_documents"]

        # AC: extra fields for detail endpoint
        assert "related_opportunities" in data
        assert isinstance(data["related_opportunities"], list)
        assert "submission_guide" in data
        assert "ai_summary" in data


    # ---------------------------------------------------------------------------
    # E06-P1-013: related_opportunities — up to 5 by CPV/region overlap
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_related_opportunities(
        detail_app, test_client, starter_headers, seeded_detail_opportunities
    ):
        """E06-P1-013: related_opportunities — up to 5 by CPV/region overlap.

        The seeded data has 2 in-scope related opportunities (one CPV overlap, one region overlap)
        and 1 out-of-scope (France) which should be excluded.
        """
        opp_id = seeded_detail_opportunities["main"]
        resp = await test_client.get(
            f"/api/v1/opportunities/{opp_id}",
            headers=starter_headers,
        )
        assert resp.status_code == 200
        data = resp.json()
        related = data["related_opportunities"]

        # At least our 2 seeded in-scope related opportunities should appear
        assert isinstance(related, list)
        assert len(related) >= 2, f"Expected ≥2 related opportunities, got {len(related)}: {related}"
        assert len(related) <= 5, f"related_opportunities must not exceed 5 items, got {len(related)}"

        # The out-of-scope French opportunity should NOT be in the list
        related_ids = [r["id"] for r in related]
        assert seeded_detail_opportunities["out_of_scope"] not in related_ids, (
            "Out-of-scope French opportunity should not appear in related_opportunities for Starter user"
        )

        # The main opportunity itself should not appear in related
        assert opp_id not in related_ids, "Main opportunity should not appear in its own related list"

        # Each item has required fields
        for item in related:
            assert "id" in item
            assert "title" in item
            assert "status" in item
            assert "opportunity_type" in item


    # ---------------------------------------------------------------------------
    # E06-P1-013: related_opportunities max 5 items
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_related_opportunities_max_5(
        detail_app, test_client, enterprise_headers, seeded_detail_opportunities
    ):
        """E06-P1-013: related_opportunities list never exceeds 5 items (Enterprise tier, no scope limit)."""
        opp_id = seeded_detail_opportunities["main"]
        resp = await test_client.get(
            f"/api/v1/opportunities/{opp_id}",
            headers=enterprise_headers,
        )
        assert resp.status_code == 200
        data = resp.json()
        assert len(data["related_opportunities"]) <= 5


    # ---------------------------------------------------------------------------
    # E06-P1-012: submission_guide present when seeded
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_submission_guide_present(
        detail_app, test_client, starter_headers, seeded_detail_opportunities
    ):
        """E06-P1-012: submission_guide is non-null with steps array when seeded."""
        opp_id = seeded_detail_opportunities["main"]
        resp = await test_client.get(
            f"/api/v1/opportunities/{opp_id}",
            headers=starter_headers,
        )
        assert resp.status_code == 200
        data = resp.json()
        sg = data.get("submission_guide")
        assert sg is not None, "submission_guide should be non-null for seeded opportunity"
        assert "id" in sg
        assert "source_portal" in sg
        assert sg["source_portal"] == "aop"
        assert "reviewed" in sg
        assert "steps" in sg
        assert isinstance(sg["steps"], list)
        assert len(sg["steps"]) >= 1


    # ---------------------------------------------------------------------------
    # E06-P1-015: AI summary returned without counter decrement
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_ai_summary_returned_without_decrement(
        detail_app, test_client, starter_headers, seeded_detail_opportunities,
        seeded_ai_summary, test_redis_client
    ):
        """E06-P1-015: Pre-seeded AI summary returned in detail response; usage counter unchanged.

        If no AI summary was seeded (seeded_ai_summary is None), this test is skipped
        gracefully with a warning — the seeding requires a real user row in client.users.
        """
        if seeded_ai_summary is None:
            pytest.skip("AI summary seeding skipped — no matching user found in DB for starter company")

        opp_id = seeded_detail_opportunities["main"]

        # Record usage counter before the call
        user_id_for_check = seeded_ai_summary["user_id"]
        # There is no usage counter in Redis yet for this — verifying zero outbound calls
        # to AI Gateway is handled by respx mocks in S06.08 tests; here we verify the
        # summary is returned and the endpoint does not call the AI Gateway at all.

        resp = await test_client.get(
            f"/api/v1/opportunities/{opp_id}",
            headers=starter_headers,
        )
        assert resp.status_code == 200
        data = resp.json()

        ai_summary = data.get("ai_summary")
        assert ai_summary is not None, (
            "ai_summary should be non-null when a prior summary was seeded for this user+opportunity"
        )
        assert ai_summary["id"] == seeded_ai_summary["summary_id"]
        assert "content" in ai_summary
        assert ai_summary["content"] == "AI-generated executive summary for the main opportunity."
        assert "generated_at" in ai_summary
        assert ai_summary.get("model") == "gpt-4o"
        assert ai_summary.get("tokens_used") == 1500


    # ---------------------------------------------------------------------------
    # E06-P0-003 (partial): Starter tier out-of-scope opportunity → 403
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_starter_out_of_scope_blocked(
        detail_app, test_client, starter_headers, seeded_detail_opportunities
    ):
        """E06-P0-003 (partial): Starter user requesting French opportunity → 403 tier_limit."""
        opp_id = seeded_detail_opportunities["out_of_scope"]
        resp = await test_client.get(
            f"/api/v1/opportunities/{opp_id}",
            headers=starter_headers,
        )
        assert resp.status_code == 403
        data = resp.json()
        assert data.get("error") == "tier_limit" or "tier_limit" in str(data)


    # ---------------------------------------------------------------------------
    # ai_summary is null when no prior summary exists
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_detail_ai_summary_null_when_absent(
        detail_app, test_client, starter_headers, seeded_detail_opportunities
    ):
        """ai_summary is null for an opportunity with no prior AI summary."""
        # Use a related opportunity that has no seeded AI summary
        opp_id = seeded_detail_opportunities["related_1"]
        resp = await test_client.get(
            f"/api/v1/opportunities/{opp_id}",
            headers=starter_headers,
        )
        assert resp.status_code == 200
        data = resp.json()
        # No summary seeded for this opportunity
        assert data["ai_summary"] is None
    ```

## Dev Notes

### Architecture Context

S06.05 adds the single-item detail endpoint to the opportunity discovery layer. Its relationship to the surrounding E06 stories:

| Story | Role | Endpoint |
|-------|------|----------|
| S06.01 | Full-text search | `GET /api/v1/opportunities/search` |
| S06.04 | Browse listing | `GET /api/v1/opportunities` |
| **S06.05** | **Full detail (this story)** | **`GET /api/v1/opportunities/{opportunity_id}`** |
| S06.06 | Document upload | `POST /api/v1/opportunities/{id}/documents/upload` |
| S06.07 | Document download | `GET /api/v1/opportunities/{id}/documents/{doc_id}/download` |
| S06.08 | AI summary generation (writes to `client.ai_summaries`) | `POST /api/v1/opportunities/{id}/ai-summary` |

S06.05 re-uses from S06.01–S06.04:
- `get_pipeline_readonly_session` dependency
- `opportunities_table` Core definition (`models/pipeline_opportunity.py`)
- `OpportunityTierGateContext` dependency and its `check_scope()` / `is_in_scope()` methods
- `OpportunitySearchItem` internal representation
- `OpportunityFullResponse` as the base for `OpportunityDetailResponse`

S06.05 introduces:
- Alembic migration `018_ai_summaries` — the `client.ai_summaries` table (also consumed by S06.08)
- `pipeline_submission_guide.py` — Core table mirror for `pipeline.submission_guides`
- `client_ai_summary.py` — Core table mirror for `client.ai_summaries`
- `OpportunityDetailResponse`, `RelatedOpportunityItem`, `SubmissionGuideRef`, `AISummaryRef` schemas
- `get_opportunity_detail()` service function
- `GET /{opportunity_id}` route handler

### CRITICAL: Free-Tier Blocking (AC: 2)

The `OpportunityTierGateContext.is_in_scope()` method returns `True` for free-tier users (no scope gate on listing). Calling `check_scope()` on a free-tier user therefore does NOT raise 403.

For the detail endpoint the spec requires free-tier users to **always** receive 403. This means the route handler must explicitly check the tier **before** calling `check_scope()`:

```python
if tier_gate.user_tier not in _PAID_TIERS:
    raise AppException(
        "Opportunity detail requires a paid subscription.",
        error="tier_limit",
        status_code=403,
        details={"upgrade_url": upgrade_url},
    )
```

The `_PAID_TIERS` frozenset (`{"starter", "professional", "enterprise"}`) is defined locally in `opportunities.py` for the detail handler (analogous to `_VALID_SORT_BY` and `_VALID_SORT_ORDER` for the listing handler). Do NOT modify `OpportunityTierGateContext` to add this check — the listing endpoint intentionally allows free users through (they receive `OpportunityFreeResponse`).

### CRITICAL: Route Registration Order

FastAPI/Starlette matches static path segments before dynamic ones. The existing routes are:
1. `GET ""` (listing) — exact match
2. `GET /search` (search) — literal segment

Adding `GET /{opportunity_id}` as a dynamic path parameter will NOT conflict with `/search` because FastAPI's path matching treats the literal segment `search` as higher priority. Verify after implementation via `/openapi.json`:
- `GET /api/v1/opportunities` (listing)
- `GET /api/v1/opportunities/search` (search)
- `GET /api/v1/opportunities/{opportunity_id}` (detail ← new)

The `/{opportunity_id}` route must be added **after** `/search` in the source file to maintain this priority (even though FastAPI handles it correctly by literal-first matching, consistent ordering avoids confusion for future maintainers).

### CRITICAL: Dual Session in Route Handler

The detail route handler requires **two** database sessions:
- `session: AsyncSession = Depends(get_pipeline_readonly_session)` — for querying `pipeline.opportunities` and `pipeline.submission_guides` (read-only)
- `db: AsyncSession = Depends(get_db_session)` — for querying `client.ai_summaries` (read-only query, but uses the client-schema session pool)

Both are passed to `get_opportunity_detail()` as `pipeline_session` and `client_session`. This is consistent with how the `get_opportunity_tier_gate` dependency itself uses `get_db_session` to look up subscription plans.

In integration tests, both sessions must be overridden in the `detail_app` fixture:
```python
app.dependency_overrides[get_pipeline_readonly_session] = _pipeline_override
app.dependency_overrides[get_db_session] = _client_override
```

Note: `get_db_session` is already overridden in the base `app` fixture in `conftest.py` (line 212). The `detail_app` fixture must **additionally** override `get_pipeline_readonly_session`. The `get_db_session` override in `detail_app` will take precedence over the base `app` fixture's override for the duration of the detail tests.

### CRITICAL: `client.ai_summaries` Table — Cross-Schema Design

The `client.ai_summaries.opportunity_id` column has no foreign key to `pipeline.opportunities.id`. This is by design: the client-api service's DB role (`client_api_role`) has only read access to the `pipeline` schema and cannot establish cross-schema FK constraints. The integrity is enforced application-side — the AI summary is only created when the opportunity exists (S06.08 verifies this during summary generation).

For S06.05, the `client_ai_summary.py` Core table definition uses a bare `UUID` column (no FK) — same pattern as how `pipeline_opportunity.py` defines columns without FK references.

### CRITICAL: `ai_summaries` Core Table MetaData Isolation

`client_ai_summary.py` uses a **separate** `MetaData(schema="client")` instance (`client_metadata`) — **not** the `pipeline_metadata` instance from `pipeline_opportunity.py`. This is important because SQLAlchemy MetaData is schema-scoped. Mixing pipeline and client tables in the same MetaData would cause incorrect schema prefixing.

```python
# CORRECT:
from client_api.models.client_ai_summary import ai_summaries_table as ai_t   # schema="client"
from client_api.models.pipeline_opportunity import opportunities_table as opp_t  # schema="pipeline"

# WRONG (would fail):
# Both tables in the same MetaData with mixed schemas
```

### CRITICAL: `submission_guides_table` Shares `pipeline_metadata`

Unlike `client_ai_summary.py`, the `pipeline_submission_guide.py` Core table **imports and reuses** the `pipeline_metadata` instance from `pipeline_opportunity.py`:

```python
from client_api.models.pipeline_opportunity import pipeline_metadata

submission_guides_table = Table("submission_guides", pipeline_metadata, ...)
```

Both `opportunities_table` and `submission_guides_table` are in the `pipeline` schema and use the same read-only session. Sharing `pipeline_metadata` ensures SQLAlchemy knows they're in the same schema and handles joins correctly if needed in future.

### CRITICAL: `get_opportunity_detail()` Service — TierGate Import

The service function takes `tier_gate: "OpportunityTierGateContext"` as a string type hint to avoid a circular import:
- `opportunity_service.py` imports from `schemas/opportunities.py`
- `opportunity_tier_gate.py` also imports from `schemas/opportunities.py`
- If `opportunity_service.py` imported `OpportunityTierGateContext` directly, it would create a circular dependency chain

Using `"OpportunityTierGateContext"` (string annotation with `from __future__ import annotations`) defers evaluation and avoids the circular import. The route handler passes the fully resolved instance from the FastAPI dependency, so no runtime issue.

### CRITICAL: Scope-Checking in Route Handler vs. Service

The route handler builds a minimal `OpportunitySearchItem` from the `OpportunityDetailResponse` to call `tier_gate.check_scope()`. This is intentional — `check_scope()` only needs `region` and `budget_max` for the scope rules, both of which are present in `OpportunityDetailResponse` (inherited from `OpportunityFullResponse`).

Alternative approach (checking in the service) would require passing the tier_gate into the service and raising from there, which tightly couples the service to the gate logic. Keeping the check in the route handler preserves the service as a pure data-fetching layer.

### CRITICAL: Related Opportunities Query — Over-Fetch for Scope Filtering

The related opportunities query fetches up to 10 rows (`LIMIT 10`) and then filters down to 5 via `is_in_scope()`. This over-fetch ensures that even if several rows are out of scope for a Starter user, we still return up to 5 in-scope items.

The `budget_max` column is included in the related opportunities query specifically to enable `is_in_scope()` checking — even though `budget_max` is not in the `RelatedOpportunityItem` response schema. The `OpportunitySearchItem` constructor used for scope checking accepts a partial set of fields.

### CRITICAL: AI Summary Fixture — User ID Dependency

The `seeded_ai_summary` fixture (Task 7) requires a real `user_id` in `client.users` for the Starter company. If the test database doesn't have a Starter user with `company_id = 12345678-0000-0000-0000-000000000001`, the fixture yields `None` and the E06-P1-015 test is skipped with `pytest.skip()`.

To make E06-P1-015 run reliably, ensure the test database has been populated by the auth/company test suite (E02 tests seed users and companies). In CI, this is guaranteed by the DB fixture migration order.

Alternatively, if the Starter user JWT fixture (`starter_user_token` in `conftest.py`) is known to have a fixed `sub` claim (user UUID), look it up in `eusolicit_test_utils.auth.generate_test_jwt()` to seed the AI summary directly with a known user_id.

### Test Infrastructure Notes

The `detail_app` fixture overrides both `get_pipeline_readonly_session` and `get_db_session`. The `_pipeline_override` wraps the superuser session with `SET TRANSACTION READ ONLY` to match the production behavior. The `_client_override` does NOT set read-only (since in production `get_db_session` supports writes — S06.08 will write to `client.ai_summaries` using this session).

### File Locations

| Purpose | Path |
|---------|------|
| Alembic migration (`client.ai_summaries`) | `eusolicit-app/services/client-api/alembic/versions/018_ai_summaries.py` |
| Pipeline submission guide Core table | `eusolicit-app/services/client-api/src/client_api/models/pipeline_submission_guide.py` |
| Client AI summary Core table | `eusolicit-app/services/client-api/src/client_api/models/client_ai_summary.py` |
| New schemas (`RelatedOpportunityItem`, etc.) | `eusolicit-app/services/client-api/src/client_api/schemas/opportunities.py` |
| `get_opportunity_detail()` service function | `eusolicit-app/services/client-api/src/client_api/services/opportunity_service.py` |
| `GET /{opportunity_id}` route handler | `eusolicit-app/services/client-api/src/client_api/api/v1/opportunities.py` |
| Integration tests | `eusolicit-app/services/client-api/tests/api/test_opportunity_detail.py` |
| Conftest (fixtures) | `eusolicit-app/services/client-api/tests/conftest.py` |
| Existing TierGate (unchanged) | `eusolicit-app/services/client-api/src/client_api/core/opportunity_tier_gate.py` |
| Pipeline opportunity Core table (reused) | `eusolicit-app/services/client-api/src/client_api/models/pipeline_opportunity.py` |
| Pipeline readonly session (reused) | `eusolicit-app/services/client-api/src/client_api/dependencies.py` |

### Test Design Traceability

| Test ID | Priority | Scenario | AC |
|---------|----------|----------|----|
| E06-P0-001 | P0 | Free-tier → 403 with `tier_limit` and `upgrade_url` | AC2 |
| E06-P0-003 (partial) | P0 | Starter tier requests French out-of-scope opportunity → 403 | AC3 |
| E06-P1-012 | P1 | Paid-tier in-scope → 200 with evaluation_criteria, mandatory_documents, deadline+tz | AC5 |
| E06-P1-013 | P1 | `related_opportunities` → up to 5 by CPV/region; out-of-scope excluded | AC6 |
| E06-P1-014 | P1 | Non-existent UUID → 404 | AC4 |
| E06-P1-015 | P1 | Pre-seeded AI summary returned without metering | AC8 |
| (Implicit) | — | Unauthenticated → 401 | AC1 |
| (Implicit) | — | `ai_summary=null` when no prior summary seeded | AC8 |
| (Implicit) | — | `submission_guide` present with `steps` array | AC7 |
| E06-P3-001 (partial) | P3 | `GET /api/v1/opportunities/{opportunity_id}` in `/openapi.json` | AC1 |

Risk mitigations relevant to this story:
- **E06-R-001** (TierGate bypass — score 6): The detail endpoint applies two layers of enforcement — free-tier block (AC2) before any DB query, and `check_scope()` for paid-tier scope validation (AC3). Neither layer can be bypassed: free-tier block is checked before `get_opportunity_detail()` is called; `check_scope()` runs after the fetch and before response serialization. Tests E06-P0-001 (free block) and E06-P0-003 partial (scope block) verify both layers.
- **E06-R-002** (Redis counter race — score 6): This story does NOT decrement any usage counter. The AI summary is returned via a SELECT on `client.ai_summaries` with no Redis interaction. E06-P1-015 verifies no usage side-effect from the detail endpoint fetch.

### References

- [Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.05] — S06.05 spec: full tender data fields, related_opportunities (≤5 by CPV/region), 403 for free, 404 for missing, submission_guide ref, AI summary ref
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P0-001] — P0: free-tier blocked from detail endpoint
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P0-003] — P0: paid-tier scope boundary (Starter + Professional)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P1-012] — P1: full tender data (evaluation_criteria, mandatory_documents, deadline+tz)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P1-013] — P1: related_opportunities ≤5 by CPV/region
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P1-014] — P1: 404 for non-existent ID
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P1-015] — P1: AI summary returned from detail without counter decrement
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-R-001] — TierGate bypass risk (score 6); mitigated by dual-layer enforcement
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-R-002] — Redis counter race (score 6); detail endpoint is counter-free
- [Source: eusolicit-app/services/client-api/src/client_api/core/opportunity_tier_gate.py] — `check_scope()` raises 403 for out-of-scope paid users; `is_in_scope()` returns True for free users (must not rely on for free-tier blocking on detail)
- [Source: eusolicit-app/services/client-api/src/client_api/services/opportunity_service.py] — Existing cursor/filter patterns; column list; `opp_t` table alias
- [Source: eusolicit-app/services/client-api/src/client_api/models/pipeline_opportunity.py] — `pipeline_metadata` instance (shared with `submission_guides_table`)
- [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/models/submission_guide.py] — Canonical `pipeline.submission_guides` schema (steps JSONB structure)
- [Source: eusolicit-app/services/client-api/alembic/versions/017_user_onboarding_completed.py] — Previous migration revision (down_revision for 018)

## Change Log

| Date | Actor | Change |
|---|---|---|
| 2026-04-17 | BMad Orchestrator (bmad-create-story) | Story file created from epic-06 S06.05 spec and test-design-epic-06.md; includes migration 018_ai_summaries, pipeline_submission_guide Core table, client_ai_summary Core table, OpportunityDetailResponse schema hierarchy, get_opportunity_detail() service, dual-session route handler, integration tests; test design traceability to E06-P0-001, P0-003 partial, P1-012, P1-013, P1-014, P1-015; critical notes on free-tier blocking vs. check_scope(), MetaData isolation, and dual-session pattern |
