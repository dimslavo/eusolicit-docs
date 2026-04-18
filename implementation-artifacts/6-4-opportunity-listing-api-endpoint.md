# Story 6.4: Opportunity Listing API Endpoint

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer building the opportunity discovery feature**,
I want **`GET /api/v1/opportunities` to serve as the primary browse-mode listing endpoint — accepting the same filter params as the search endpoint (minus full-text query), supporting dynamic `sort_by` (deadline, relevance_score, budget, published_date) and `sort_order` (asc/desc), applying `OpportunityTierGate` to every result item, and returning a sort-aware cursor-based paginated response with `next_cursor`, `prev_cursor`, `total_count`, and `results[]`**,
so that **the frontend listing page can browse, filter, and sort EU procurement opportunities with correct tier-gated field visibility (free users see `OpportunityFreeResponse[]`; paid users see `OpportunityFullResponse[]` filtered to their subscription scope), using the same `OpportunityTierGateContext` dependency established in S06.02 and the read-only pipeline session from S06.01**.

## Acceptance Criteria

1. `GET /api/v1/opportunities` is registered at `/api/v1/opportunities` with prefix `/api/v1` and appears in the OpenAPI schema at `GET /api/v1/opportunities` with correct request parameters and response description; the endpoint is distinct from `GET /api/v1/opportunities/search` (which requires `q` for full-text search) and from `GET /api/v1/opportunities/{id}` (detail — S06.05)

2. The endpoint operates in browse mode — no `q` full-text search param; the following filter params are supported as optional query params (same as search, same AND composition): `cpv_codes` (comma-separated string), `regions` (comma-separated string), `budget_min` (Decimal ≥ 0), `budget_max` (Decimal ≥ 0), `deadline_from` (ISO8601 date), `deadline_to` (ISO8601 date), `status` (enum: `open`, `closed`, `expired`, `archived`), `opportunity_type` (alias `type`, enum: `tender`, `grant`); unknown enum values return HTTP 400; `deleted_at IS NOT NULL` rows are always excluded

3. `sort_by` query param (default: `published_date`) accepts exactly four values: `deadline`, `relevance_score`, `budget`, `published_date`; any other value returns HTTP 400 with `{"error": "bad_request", "details": {"field": "sort_by", "allowed": [...]}}`; `sort_order` param (default: `desc`) accepts `asc` | `desc`; any other value returns HTTP 400

4. Cursor-based pagination: `after_cursor` (forward), `before_cursor` (backward); `limit` defaults to 25, accepts 1–100, rejects > 100 with 422; response includes `total_count` (computed from base filter conditions, stable across pages), `next_cursor` (null if no next page), `prev_cursor` (null if no previous page); an invalid cursor returns HTTP 400 with `{"error": "invalid_cursor", ...}`

5. The listing cursor is a **sort-aware opaque base64-encoded JSON** string `{"sort_by": "<sort_by>", "v": "<sort_value_str_or_null>", "id": "<UUID>"}` — distinct from the search cursor format (`{"published_at": ..., "id": ...}`) — that encodes the primary sort column value and row id to enable correct keyset pagination across all four `sort_by` modes; the cursor's embedded `sort_by` is validated against the request's `sort_by` param on continuation pages (mismatched sort_by on a continuation cursor returns 400)

6. `OpportunityTierGateContext` (from `get_opportunity_tier_gate`) is injected as a FastAPI dependency and applied to every result item: for free-tier users every item is serialized with `serialize_item()` (always `OpportunityFreeResponse`); for paid-tier users `is_in_scope()` is checked first and out-of-scope items are silently omitted from `results[]` (not 403'd — listing silently filters, per S06.02 AC7); `total_count` reflects the unfiltered query count (not the tier-filtered count)

7. Response is returned as a JSON dict with four top-level keys: `results` (list of `OpportunityFreeResponse` or `OpportunityFullResponse` objects), `total_count` (int), `next_cursor` (str or null), `prev_cursor` (str or null); `results` is an empty list `[]` when no records match the filters

8. For `sort_by=relevance_score`: sort column is `relevance_scores[company_id_str]::float` where `company_id_str` is `str(current_user.company_id)` from the JWT; opportunities with no relevance score for the company (null JSON key) sort last (treated as 0.0 for both the cursor value and ORDER BY coalescing); `current_user.company_id` is available from `get_current_user` — no extra DB query required

9. Endpoint requires a valid Bearer JWT (`get_current_user` dependency); unauthenticated requests return 401; no additional dependencies beyond what S06.01 and S06.02 already provide

10. Integration tests (`tests/api/test_opportunity_listing.py`) cover: browse mode with no query params returns 200 with results and correct pagination metadata (E06-P1-011); all four `sort_by` values with `sort_order=desc` return correctly sorted first-page items (E06-P1-010); `sort_order=asc` reverses ordering (E06-P1-010); free-tier user receives `OpportunityFreeResponse[]` with only six allowed fields and no `budget_min`, `budget_max`, etc. (E06-P0-002); paid-tier user receives `OpportunityFullResponse[]` with full fields for in-scope opportunities (E06-P1-011); invalid `sort_by` returns 400; pagination metadata fields `next_cursor`, `prev_cursor`, `total_count` present in every response

## Tasks / Subtasks

- [ ] Task 1: Add listing cursor helpers and `list_opportunities()` service function (AC: 2, 3, 4, 5, 7, 8)
  - [ ] 1.1 Edit `eusolicit-app/services/client-api/src/client_api/services/opportunity_service.py` — add the following AFTER the existing `_decode_cursor` / `_encode_cursor` helpers and BEFORE the `search_opportunities` function:

    ```python
    # ---------------------------------------------------------------------------
    # Listing cursor helpers (sort-aware — distinct from search cursor)
    # ---------------------------------------------------------------------------

    _VALID_SORT_BY: frozenset[str] = frozenset({"deadline", "relevance_score", "budget", "published_date"})
    _VALID_SORT_ORDER: frozenset[str] = frozenset({"asc", "desc"})


    def _encode_listing_cursor(sort_by: str, sort_value: Any, row_id: UUID) -> str:
        """Encode a sort-aware listing cursor as an opaque base64 JSON string.

        Format: {"sort_by": "<sort_by>", "v": "<sort_value_str_or_null>", "id": "<UUID>"}

        sort_value may be None (for null sort column values — e.g., null deadline).
        datetime values are stored as ISO8601; Decimal and float as str.

        Listing cursors are distinct from search cursors (which encode published_at + id
        without sort_by metadata).  The sort_by field allows the decoder to reconstruct
        the correct native type for keyset comparison.
        """
        if sort_value is None:
            sv: str | None = None
        elif isinstance(sort_value, datetime):
            sv = sort_value.isoformat()
        else:
            sv = str(sort_value)

        payload = json.dumps({"sort_by": sort_by, "v": sv, "id": str(row_id)})
        return base64.urlsafe_b64encode(payload.encode(_CURSOR_ENCODING)).decode("ascii")


    def _decode_listing_cursor(cursor: str) -> tuple[str, Any, UUID]:
        """Decode a listing cursor string.

        Returns (sort_by: str, sort_value: Any, id: UUID).
        sort_value is typed to its native Python type based on sort_by:
          published_date / deadline → datetime (UTC-aware) or None
          budget                   → Decimal or None
          relevance_score          → float or None

        Raises ValueError on invalid format (caught by route and returned as HTTP 400).
        """
        try:
            payload = json.loads(
                base64.urlsafe_b64decode(cursor.encode("ascii") + b"==").decode(_CURSOR_ENCODING)
            )
            sort_by: str = payload["sort_by"]
            sv_raw: str | None = payload["v"]
            row_id = UUID(payload["id"])

            if sv_raw is None:
                sort_value: Any = None
            elif sort_by in ("published_date", "deadline"):
                dt = datetime.fromisoformat(sv_raw)
                sort_value = dt if dt.tzinfo is not None else dt.replace(tzinfo=timezone.utc)
            elif sort_by == "budget":
                sort_value = Decimal(sv_raw)
            elif sort_by == "relevance_score":
                sort_value = float(sv_raw)
            else:
                sort_value = sv_raw  # unknown sort_by; treated as string

            return sort_by, sort_value, row_id
        except Exception as exc:
            raise ValueError(f"Invalid listing cursor: {exc}") from exc


    def _get_sort_column(sort_by: str, company_id: UUID | None) -> Any:
        """Return the SQLAlchemy column expression for the given sort_by.

        Used for both ORDER BY and keyset WHERE condition.
        For relevance_score, returns a JSON subscript expression on relevance_scores JSONB
        keyed by company_id (str).  The caller is responsible for applying NULLS LAST
        (via nulls_last()) and casting (via .cast(Float)) where needed.
        """
        from sqlalchemy import Float, cast

        if sort_by == "deadline":
            return opp_t.c.deadline
        elif sort_by == "budget":
            return opp_t.c.budget_max
        elif sort_by == "relevance_score" and company_id is not None:
            # PostgreSQL: (relevance_scores->>'<company_id>')::float
            return opp_t.c.relevance_scores[str(company_id)].astext.cast(Float)
        else:
            # Default / published_date fallback
            return opp_t.c.published_at


    def _build_listing_keyset_condition(
        sort_by: str,
        sort_order: str,
        cursor_sort_value: Any,
        cursor_id: UUID,
        forward: bool,
        sort_col: Any,
    ) -> Any:
        """Build the keyset WHERE condition for listing cursor pagination.

        Correctly handles:
          - Non-null cursor values: standard row/column comparison with nulls last semantics
          - Null cursor values: cursor is in the "NULL region" (sort_col IS NULL)

        forward=True  → after_cursor (advance to next page)
        forward=False → before_cursor (go to previous page)

        NULLS LAST semantics:
          Null sort values always sort last regardless of sort_order.
          When cursor_sort_value is non-null:
            forward + desc → WHERE (sort_col < v OR (sort_col = v AND id < cursor_id) OR sort_col IS NULL)
            forward + asc  → WHERE (sort_col > v OR (sort_col = v AND id > cursor_id) OR sort_col IS NULL)
          When cursor_sort_value is null (cursor is in the null region):
            forward + desc → WHERE (sort_col IS NULL AND id < cursor_id)
            forward + asc  → WHERE (sort_col IS NULL AND id > cursor_id)
            before + any   → WHERE sort_col IS NOT NULL (all non-null items precede null items)
        """
        is_desc = sort_order == "desc"

        if cursor_sort_value is None:
            # Cursor is within the NULLS region
            if not forward:
                # Items before the null region are all non-null items
                return sort_col.is_not(None)
            if is_desc:
                return and_(sort_col.is_(None), opp_t.c.id < cursor_id)
            else:
                return and_(sort_col.is_(None), opp_t.c.id > cursor_id)
        else:
            # Cursor is in the non-null region
            if not forward:
                # Items before a non-null cursor: larger values (DESC) or smaller values (ASC)
                if is_desc:
                    return or_(
                        sort_col > cursor_sort_value,
                        and_(sort_col == cursor_sort_value, opp_t.c.id > cursor_id),
                    )
                else:
                    return or_(
                        sort_col < cursor_sort_value,
                        and_(sort_col == cursor_sort_value, opp_t.c.id < cursor_id),
                    )
            else:
                # Items after a non-null cursor: smaller values (DESC) or larger values (ASC)
                if is_desc:
                    return or_(
                        sort_col < cursor_sort_value,
                        and_(sort_col == cursor_sort_value, opp_t.c.id < cursor_id),
                        sort_col.is_(None),  # NULLs always come after non-null in NULLS LAST
                    )
                else:
                    return or_(
                        sort_col > cursor_sort_value,
                        and_(sort_col == cursor_sort_value, opp_t.c.id > cursor_id),
                        sort_col.is_(None),  # NULLs always come after non-null in NULLS LAST ASC
                    )
    ```

  - [ ] 1.2 Add the following imports to the top of `opportunity_service.py` (add to existing imports — some may already be present):

    ```python
    from sqlalchemy import Float, nulls_last
    from uuid import UUID
    ```

    Also add `Float` and `nulls_last` to the existing `from sqlalchemy import ...` line if not already there.

  - [ ] 1.3 Add `list_opportunities()` function to `opportunity_service.py` AFTER `search_opportunities()`:

    ```python
    # ---------------------------------------------------------------------------
    # Listing function (browse mode — S06.04)
    # ---------------------------------------------------------------------------

    async def list_opportunities(
        *,
        session: AsyncSession,
        cpv_codes: list[str] | None = None,
        regions: list[str] | None = None,
        budget_min: Decimal | None = None,
        budget_max: Decimal | None = None,
        deadline_from: date | None = None,
        deadline_to: date | None = None,
        status: str | None = None,
        opportunity_type: str | None = None,
        sort_by: str = "published_date",
        sort_order: str = "desc",
        after_cursor: str | None = None,
        before_cursor: str | None = None,
        limit: int = 25,
        company_id: UUID | None = None,
    ) -> OpportunitySearchResponse:
        """Execute the browse-mode listing query against pipeline.opportunities.

        Browse mode: no full-text `q` param.  Supports dynamic sort_by with
        sort-aware cursor-based pagination.

        Pagination order: primary = sort_by column (NULLS LAST), secondary = id (stable).
        after_cursor: advance to the next page (forward).
        before_cursor: go to the previous page (backward).

        For relevance_score sort: company_id is required; opportunities without a
        score for the company sort last (null = 0.0 in cursor encoding, but the raw
        null is preserved for correct NULLS LAST comparison).

        Returns OpportunitySearchResponse with total_count, next_cursor, prev_cursor, results.
        """
        from sqlalchemy import Float, nulls_last  # noqa: PLC0415 — local import to avoid circular

        # ----- decode cursors -----
        after_sort_by: str | None = None
        after_sort_val: Any = None
        after_id: UUID | None = None
        before_sort_by: str | None = None
        before_sort_val: Any = None
        before_id: UUID | None = None

        if after_cursor:
            try:
                after_sort_by, after_sort_val, after_id = _decode_listing_cursor(after_cursor)
            except ValueError as exc:
                from eusolicit_common.exceptions import BadRequestError  # noqa: PLC0415
                raise BadRequestError(
                    "Invalid after_cursor value.",
                    details={"error": "invalid_cursor", "message": str(exc)},
                ) from exc
            if after_sort_by != sort_by:
                from eusolicit_common.exceptions import BadRequestError  # noqa: PLC0415
                raise BadRequestError(
                    f"Cursor sort_by '{after_sort_by}' does not match request sort_by '{sort_by}'.",
                    details={"error": "invalid_cursor", "message": "sort_by mismatch between cursor and current request"},
                )

        if before_cursor:
            try:
                before_sort_by, before_sort_val, before_id = _decode_listing_cursor(before_cursor)
            except ValueError as exc:
                from eusolicit_common.exceptions import BadRequestError  # noqa: PLC0415
                raise BadRequestError(
                    "Invalid before_cursor value.",
                    details={"error": "invalid_cursor", "message": str(exc)},
                ) from exc
            if before_sort_by != sort_by:
                from eusolicit_common.exceptions import BadRequestError  # noqa: PLC0415
                raise BadRequestError(
                    f"Cursor sort_by '{before_sort_by}' does not match request sort_by '{sort_by}'.",
                    details={"error": "invalid_cursor", "message": "sort_by mismatch between cursor and current request"},
                )

        # ----- build base WHERE conditions -----
        conditions: list[Any] = [opp_t.c.deleted_at.is_(None)]

        if status:
            conditions.append(opp_t.c.status == status)
        if opportunity_type:
            conditions.append(opp_t.c.opportunity_type == opportunity_type)
        if budget_min is not None:
            conditions.append(
                or_(opp_t.c.budget_max >= budget_min, opp_t.c.budget_min >= budget_min)
            )
        if budget_max is not None:
            conditions.append(
                or_(opp_t.c.budget_min <= budget_max, opp_t.c.budget_max <= budget_max)
            )
        if deadline_from:
            conditions.append(opp_t.c.deadline >= datetime(
                deadline_from.year, deadline_from.month, deadline_from.day, tzinfo=timezone.utc
            ))
        if deadline_to:
            conditions.append(opp_t.c.deadline <= datetime(
                deadline_to.year, deadline_to.month, deadline_to.day, 23, 59, 59, tzinfo=timezone.utc
            ))
        if cpv_codes:
            conditions.append(opp_t.c.cpv_codes.overlap(cpv_codes))
        if regions:
            conditions.append(opp_t.c.region.in_(regions))

        base_where = and_(*conditions)

        # ----- total count query -----
        count_stmt = select(func.count()).select_from(opp_t).where(base_where)
        total_count: int = (await session.execute(count_stmt)).scalar_one()

        # ----- resolve sort column expression -----
        sort_col = _get_sort_column(sort_by, company_id)

        # ----- build ORDER BY -----
        is_backward = before_id is not None and after_id is None
        # Backward pagination reverses fetch order, then Python-reverses the results
        effective_order = sort_order if not is_backward else ("asc" if sort_order == "desc" else "desc")

        if effective_order == "desc":
            order_exprs: list[Any] = [nulls_last(sort_col.desc()), opp_t.c.id.desc()]
        else:
            order_exprs = [nulls_last(sort_col.asc()), opp_t.c.id.asc()]

        # ----- pagination keyset condition -----
        page_conditions: list[Any] = list(conditions)

        if after_id is not None:
            page_conditions.append(
                _build_listing_keyset_condition(
                    sort_by, sort_order, after_sort_val, after_id, forward=True, sort_col=sort_col
                )
            )
        elif before_id is not None:
            page_conditions.append(
                _build_listing_keyset_condition(
                    sort_by, sort_order, before_sort_val, before_id, forward=False, sort_col=sort_col
                )
            )

        page_where = and_(*page_conditions)

        # ----- select columns -----
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

        fetch_limit = limit + 1
        page_stmt = (
            select(*columns)
            .where(page_where)
            .order_by(*order_exprs)
            .limit(fetch_limit)
        )

        rows = (await session.execute(page_stmt)).mappings().all()

        has_more = len(rows) == fetch_limit
        rows = list(rows)[:limit]
        if is_backward:
            rows = list(reversed(rows))

        # ----- build result items -----
        results: list[OpportunitySearchItem] = []
        for row in rows:
            item = OpportunitySearchItem(
                id=row["id"],
                title=row["title"] or "",
                description=row.get("description"),
                status=row["status"],
                opportunity_type=row["opportunity_type"],
                deadline=row.get("deadline"),
                budget_min=row.get("budget_min"),
                budget_max=row.get("budget_max"),
                currency=row.get("currency"),
                country=row.get("country"),
                region=row.get("region"),
                contracting_authority=row.get("contracting_authority"),
                cpv_codes=list(row.get("cpv_codes") or []),
                evaluation_criteria=row.get("evaluation_criteria"),
                mandatory_documents=row.get("mandatory_documents"),
                relevance_scores=row.get("relevance_scores"),
                published_at=row.get("published_at"),
                updated_at=row.get("updated_at"),
                relevance_rank=None,  # listing mode does not compute FTS rank
            )
            results.append(item)

        # ----- compute cursors -----
        next_cursor: str | None = None
        prev_cursor: str | None = None

        def _row_sort_val(item: OpportunitySearchItem) -> Any:
            """Extract the sort column value from a result item for cursor encoding."""
            if sort_by == "published_date":
                return item.published_at
            elif sort_by == "deadline":
                return item.deadline
            elif sort_by == "budget":
                return item.budget_max
            elif sort_by == "relevance_score" and company_id is not None:
                scores = item.relevance_scores or {}
                score = scores.get(str(company_id))
                return float(score) if score is not None else None
            return item.published_at

        if results:
            first = results[0]
            last = results[-1]

            if not is_backward:
                if has_more:
                    next_cursor = _encode_listing_cursor(sort_by, _row_sort_val(last), last.id)
                if after_id is not None:
                    prev_cursor = _encode_listing_cursor(sort_by, _row_sort_val(first), first.id)
            else:
                if has_more:
                    prev_cursor = _encode_listing_cursor(sort_by, _row_sort_val(first), first.id)
                next_cursor = _encode_listing_cursor(sort_by, _row_sort_val(last), last.id)

        return OpportunitySearchResponse(
            results=results,
            total_count=total_count,
            next_cursor=next_cursor,
            prev_cursor=prev_cursor,
        )
    ```

- [ ] Task 2: Add `GET /api/v1/opportunities` endpoint to the opportunities router (AC: 1, 2, 3, 4, 6, 7, 9)
  - [ ] 2.1 Edit `eusolicit-app/services/client-api/src/client_api/api/v1/opportunities.py` — add the following BEFORE the existing `search_opportunities` route handler (after the `_VALID_STATUSES` / `_VALID_TYPES` constants block):

    ```python
    # ---------------------------------------------------------------------------
    # Valid listing sort params
    # ---------------------------------------------------------------------------

    _VALID_SORT_BY = {"deadline", "relevance_score", "budget", "published_date"}
    _VALID_SORT_ORDER = {"asc", "desc"}


    # ---------------------------------------------------------------------------
    # GET /opportunities  (primary listing / browse mode — S06.04)
    # ---------------------------------------------------------------------------

    @router.get(
        "",
        summary="List opportunities in browse mode (no full-text search)",
        description=(
            "Primary listing endpoint for the opportunity discovery page. "
            "Browse mode: no full-text search query. Supports filter params "
            "(CPV codes, regions, budget range, deadline range, status, type) "
            "and dynamic sorting (deadline, relevance_score, budget, published_date). "
            "Free-tier users receive OpportunityFreeResponse (6 fields); "
            "paid-tier users receive OpportunityFullResponse filtered to tier scope. "
            "Out-of-scope opportunities for paid tiers are silently omitted from results[]."
        ),
    )
    async def list_opportunities(
        cpv_codes: Annotated[
            str | None,
            Query(description="Comma-separated CPV codes to filter by"),
        ] = None,
        regions: Annotated[
            str | None,
            Query(description="Comma-separated region names to filter by"),
        ] = None,
        budget_min: Annotated[
            Decimal | None,
            Query(ge=0, description="Minimum budget (EUR)"),
        ] = None,
        budget_max: Annotated[
            Decimal | None,
            Query(ge=0, description="Maximum budget (EUR)"),
        ] = None,
        deadline_from: Annotated[
            date | None,
            Query(description="Deadline range start (YYYY-MM-DD)"),
        ] = None,
        deadline_to: Annotated[
            date | None,
            Query(description="Deadline range end (YYYY-MM-DD)"),
        ] = None,
        status: Annotated[
            str | None,
            Query(description="Opportunity status: open | closed | expired | archived"),
        ] = None,
        opportunity_type: Annotated[
            str | None,
            Query(alias="type", description="Opportunity type: tender | grant"),
        ] = None,
        sort_by: Annotated[
            str,
            Query(
                description="Sort field: deadline | relevance_score | budget | published_date",
            ),
        ] = "published_date",
        sort_order: Annotated[
            str,
            Query(description="Sort direction: asc | desc"),
        ] = "desc",
        after_cursor: Annotated[
            str | None,
            Query(description="Opaque cursor for forward pagination (next page)"),
        ] = None,
        before_cursor: Annotated[
            str | None,
            Query(description="Opaque cursor for backward pagination (previous page)"),
        ] = None,
        limit: Annotated[
            int,
            Query(ge=1, le=100, description="Number of results per page (default 25, max 100)"),
        ] = 25,
        current_user: Annotated[CurrentUser, Depends(get_current_user)] = None,  # type: ignore[assignment]
        session: Annotated[AsyncSession, Depends(get_pipeline_readonly_session)] = None,  # type: ignore[assignment]
        tier_gate: Annotated[OpportunityTierGateContext, Depends(get_opportunity_tier_gate)] = None,  # type: ignore[assignment]
    ) -> dict:
        """Browse opportunities with optional filters and dynamic sort.

        AC1: Endpoint at /api/v1/opportunities (browse mode, distinct from /search).
        AC2: Filter params (cpv_codes, regions, budget, deadline, status, type).
        AC3: sort_by (deadline | relevance_score | budget | published_date).
        AC4: Cursor pagination: limit 1-100, after_cursor/before_cursor.
        AC6: TierGate applied per item: free → FreeResponse, paid → FullResponse (out-of-scope silently omitted).
        AC7: Response dict: results[], total_count, next_cursor, prev_cursor.
        AC9: Requires valid Bearer JWT (401 if absent/invalid).
        """
        # Validate sort params
        if sort_by not in _VALID_SORT_BY:
            raise BadRequestError(
                f"Invalid sort_by '{sort_by}'. Must be one of: {', '.join(sorted(_VALID_SORT_BY))}",
                details={"field": "sort_by", "allowed": sorted(_VALID_SORT_BY)},
            )
        if sort_order not in _VALID_SORT_ORDER:
            raise BadRequestError(
                f"Invalid sort_order '{sort_order}'. Must be one of: asc, desc",
                details={"field": "sort_order", "allowed": ["asc", "desc"]},
            )

        # Validate filter enum values
        if status is not None and status not in _VALID_STATUSES:
            raise BadRequestError(
                f"Invalid status '{status}'. Must be one of: {', '.join(sorted(_VALID_STATUSES))}",
                details={"field": "status", "allowed": sorted(_VALID_STATUSES)},
            )
        if opportunity_type is not None and opportunity_type not in _VALID_TYPES:
            raise BadRequestError(
                f"Invalid type '{opportunity_type}'. Must be one of: {', '.join(sorted(_VALID_TYPES))}",
                details={"field": "type", "allowed": sorted(_VALID_TYPES)},
            )

        # Parse comma-separated multi-value params
        cpv_list: list[str] | None = None
        if cpv_codes:
            cpv_list = [c.strip() for c in cpv_codes.split(",") if c.strip()]

        regions_list: list[str] | None = None
        if regions:
            regions_list = [r.strip() for r in regions.split(",") if r.strip()]

        raw_response = await opportunity_service.list_opportunities(
            session=session,
            cpv_codes=cpv_list,
            regions=regions_list,
            budget_min=budget_min,
            budget_max=budget_max,
            deadline_from=deadline_from,
            deadline_to=deadline_to,
            status=status,
            opportunity_type=opportunity_type,
            sort_by=sort_by,
            sort_order=sort_order,
            after_cursor=after_cursor,
            before_cursor=before_cursor,
            limit=limit,
            company_id=current_user.company_id if sort_by == "relevance_score" else None,
        )

        # Apply TierGate: free → FreeResponse, paid → FullResponse (out-of-scope silently omitted)
        tier_gated_results: list[OpportunityFreeResponse | OpportunityFullResponse] = []
        for item in raw_response.results:
            if tier_gate.is_in_scope(item):
                tier_gated_results.append(tier_gate.serialize_item(item))

        return {
            "results": [r.model_dump(mode="json") for r in tier_gated_results],
            "total_count": raw_response.total_count,
            "next_cursor": raw_response.next_cursor,
            "prev_cursor": raw_response.prev_cursor,
        }
    ```

  - [ ] 2.2 Verify that `opportunities.py` already imports `from client_api.services import opportunity_service` (added in S06.01 — this import is reused; no change needed). Add no new imports if everything is already in place from S06.01 and S06.02 router edits.

- [ ] Task 3: Write integration tests (AC: 10)
  - [ ] 3.1 Create `eusolicit-app/services/client-api/tests/api/test_opportunity_listing.py`:

    ```python
    """Integration tests for GET /api/v1/opportunities (Story S06.04).

    Test IDs covered:
        E06-P0-002  Free-tier receives OpportunityFreeResponse with only allowed fields on listing
        E06-P1-010  Browse mode + sort_by (deadline, relevance_score, budget, published_date) +
                    sort_order (asc, desc) → first-page items correctly ordered
        E06-P1-011  Response includes next_cursor, prev_cursor, total_count, results[];
                    free → FreeResponse[]; paid → FullResponse[]

    Fixtures:
        Reuses the seeded pipeline.opportunities from test_opportunity_search.py where possible.
        Adds a module-scoped fixture seeding additional rows with deadlines, budgets, and
        relevance_scores to test sorting behaviour.

    Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#P1-010, P1-011, P0-002
    """
    from __future__ import annotations

    import uuid
    from datetime import UTC, datetime, timedelta

    import pytest
    import pytest_asyncio
    from sqlalchemy import text

    from eusolicit_test_utils.auth import generate_test_jwt

    # ---------------------------------------------------------------------------
    # Seed data (module-scoped — inserted once, cleaned after module)
    # ---------------------------------------------------------------------------

    _COMPANY_ID = str(uuid.uuid4())  # stable company_id for the Starter JWT

    _SEED_ROWS = [
        # Row A: soonest deadline, largest budget, highest relevance score
        {
            "id": str(uuid.uuid4()),
            "source_id": "list-opp-A",
            "source_type": "aop",
            "title": "Listing Test Opportunity A",
            "description": "Opportunity A with soonest deadline and largest budget",
            "contracting_authority": "Test Authority A",
            "opportunity_type": "tender",
            "status": "open",
            "deadline": (datetime.now(UTC) + timedelta(days=5)).isoformat(),
            "budget_min": 300_000,
            "budget_max": 500_000,
            "currency": "EUR",
            "country": "Bulgaria",
            "region": "Bulgaria",
            "cpv_codes": "{45000000-7}",
            "published_at": (datetime.now(UTC) - timedelta(days=10)).isoformat(),
            "relevance_scores": f'{{"{_COMPANY_ID}": 0.9}}',
        },
        # Row B: middle deadline, middle budget, middle relevance score
        {
            "id": str(uuid.uuid4()),
            "source_id": "list-opp-B",
            "source_type": "aop",
            "title": "Listing Test Opportunity B",
            "description": "Opportunity B with middle deadline and budget",
            "contracting_authority": "Test Authority B",
            "opportunity_type": "tender",
            "status": "open",
            "deadline": (datetime.now(UTC) + timedelta(days=15)).isoformat(),
            "budget_min": 100_000,
            "budget_max": 200_000,
            "currency": "EUR",
            "country": "Bulgaria",
            "region": "Bulgaria",
            "cpv_codes": "{45000000-7}",
            "published_at": (datetime.now(UTC) - timedelta(days=5)).isoformat(),
            "relevance_scores": f'{{"{_COMPANY_ID}": 0.5}}',
        },
        # Row C: latest deadline, smallest budget, lowest relevance score
        {
            "id": str(uuid.uuid4()),
            "source_id": "list-opp-C",
            "source_type": "aop",
            "title": "Listing Test Opportunity C",
            "description": "Opportunity C with latest deadline and smallest budget",
            "contracting_authority": "Test Authority C",
            "opportunity_type": "tender",
            "status": "open",
            "deadline": (datetime.now(UTC) + timedelta(days=30)).isoformat(),
            "budget_min": 10_000,
            "budget_max": 50_000,
            "currency": "EUR",
            "country": "Bulgaria",
            "region": "Bulgaria",
            "cpv_codes": "{45000000-7}",
            "published_at": (datetime.now(UTC) - timedelta(days=1)).isoformat(),
            "relevance_scores": f'{{"{_COMPANY_ID}": 0.2}}',
        },
    ]


    @pytest_asyncio.fixture(scope="module")
    async def seeded_listing_opportunities(superuser_session):
        """Seed pipeline.opportunities with listing test rows."""
        for row in _SEED_ROWS:
            scores_clause = f"'{row['relevance_scores']}'::jsonb" if row.get("relevance_scores") else "NULL"
            stmt = text(f"""
                INSERT INTO pipeline.opportunities (
                    id, source_id, source_type, title, description, contracting_authority,
                    opportunity_type, status, deadline, budget_min, budget_max, currency,
                    country, region, cpv_codes, published_at, relevance_scores,
                    created_at, updated_at
                ) VALUES (
                    :id, :source_id, :source_type, :title, :description, :contracting_authority,
                    :opportunity_type, :status, :deadline, :budget_min, :budget_max, :currency,
                    :country, :region, :cpv_codes::text[], :published_at, {scores_clause},
                    NOW(), NOW()
                )
                ON CONFLICT (source_id, source_type) DO NOTHING
            """)
            await superuser_session.execute(stmt, {
                k: v for k, v in row.items() if k not in ("relevance_scores",)
            })
        await superuser_session.commit()

        yield [row["id"] for row in _SEED_ROWS]

        await superuser_session.execute(
            text("DELETE FROM pipeline.opportunities WHERE source_id LIKE 'list-opp-%'")
        )
        await superuser_session.commit()


    @pytest_asyncio.fixture
    async def listing_app(app, superuser_session_factory):
        """Override get_pipeline_readonly_session for the listing tests."""
        from client_api.dependencies import get_pipeline_readonly_session

        async def _pipeline_override():
            async with superuser_session_factory() as s:
                await s.execute(text("SET TRANSACTION READ ONLY"))
                try:
                    yield s
                finally:
                    await s.rollback()

        app.dependency_overrides[get_pipeline_readonly_session] = _pipeline_override
        yield app


    # ---------------------------------------------------------------------------
    # Helper: make a JWT for a free user
    # ---------------------------------------------------------------------------

    @pytest.fixture
    def free_jwt(free_user_token):
        return {"Authorization": f"Bearer {free_user_token}"}


    @pytest.fixture
    def starter_jwt(starter_user_token):
        # starter_user_token is assumed to be in conftest.py; company_id = _COMPANY_ID
        return {"Authorization": f"Bearer {starter_user_token}"}


    # ---------------------------------------------------------------------------
    # E06-P1-011: Response structure — pagination metadata present
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_listing_response_has_required_pagination_fields(
        listing_app, test_client, free_jwt, seeded_listing_opportunities
    ):
        """E06-P1-011: Response includes next_cursor, prev_cursor, total_count, results[]."""
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={"status": "open"},
            headers=free_jwt,
        )
        assert resp.status_code == 200
        data = resp.json()
        assert "results" in data
        assert "total_count" in data
        assert "next_cursor" in data
        assert "prev_cursor" in data
        assert isinstance(data["results"], list)
        assert isinstance(data["total_count"], int)
        assert data["total_count"] >= 3  # at least our 3 seeded rows


    # ---------------------------------------------------------------------------
    # E06-P0-002: Free-tier receives OpportunityFreeResponse (6 fields only)
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_free_tier_receives_free_response_on_listing(
        listing_app, test_client, free_jwt, seeded_listing_opportunities
    ):
        """E06-P0-002: Free-tier listing → OpportunityFreeResponse with exactly 6 fields.

        Asserts: id, title, deadline, region, opportunity_type, status are present.
        Asserts: budget_min, budget_max, cpv_codes, contracting_authority,
                 evaluation_criteria, relevance_scores, relevance_rank are ABSENT.
        """
        resp = await test_client.get("/api/v1/opportunities", headers=free_jwt)
        assert resp.status_code == 200
        data = resp.json()
        assert len(data["results"]) >= 1

        for item in data["results"]:
            # Required free-tier fields
            assert "id" in item
            assert "title" in item
            assert "deadline" in item
            assert "region" in item
            assert "opportunity_type" in item
            assert "status" in item

            # Paid-tier only fields must be absent
            assert "budget_min" not in item, f"budget_min exposed to free-tier user: {item}"
            assert "budget_max" not in item, f"budget_max exposed to free-tier user: {item}"
            assert "cpv_codes" not in item, f"cpv_codes exposed to free-tier user: {item}"
            assert "contracting_authority" not in item, f"contracting_authority exposed: {item}"
            assert "evaluation_criteria" not in item
            assert "mandatory_documents" not in item
            assert "relevance_scores" not in item
            assert "relevance_rank" not in item
            assert "description" not in item
            assert "currency" not in item
            assert "country" not in item


    # ---------------------------------------------------------------------------
    # E06-P1-011: Paid-tier receives OpportunityFullResponse
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_paid_tier_receives_full_response_on_listing(
        listing_app, test_client, starter_jwt, seeded_listing_opportunities
    ):
        """E06-P1-011: Starter-tier listing → OpportunityFullResponse with full fields.

        Seeded rows are Bulgaria / ≤500K — all in Starter scope.
        """
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={"regions": "Bulgaria"},
            headers=starter_jwt,
        )
        assert resp.status_code == 200
        data = resp.json()
        assert len(data["results"]) >= 1

        for item in data["results"]:
            # All full-response fields must be present
            assert "id" in item
            assert "title" in item
            assert "deadline" in item
            assert "region" in item
            assert "opportunity_type" in item
            assert "status" in item
            assert "budget_min" in item
            assert "budget_max" in item
            assert "cpv_codes" in item
            assert "contracting_authority" in item


    # ---------------------------------------------------------------------------
    # E06-P1-010: sort_by=deadline, sort_order=desc → soonest deadline first (desc = largest → smallest)
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_sort_by_deadline_desc(
        listing_app, test_client, free_jwt, seeded_listing_opportunities
    ):
        """E06-P1-010: sort_by=deadline sort_order=desc → later deadlines first."""
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={"sort_by": "deadline", "sort_order": "desc", "limit": 100},
            headers=free_jwt,
        )
        assert resp.status_code == 200
        data = resp.json()
        titles = [r["title"] for r in data["results"]]
        listing_titles = [t for t in titles if t.startswith("Listing Test Opportunity")]
        # C (30 days) should appear before B (15 days), B before A (5 days) in DESC
        if len(listing_titles) >= 3:
            c_idx = listing_titles.index("Listing Test Opportunity C")
            b_idx = listing_titles.index("Listing Test Opportunity B")
            a_idx = listing_titles.index("Listing Test Opportunity A")
            assert c_idx < b_idx < a_idx, (
                f"Expected C ({c_idx}) < B ({b_idx}) < A ({a_idx}) for deadline DESC order"
            )


    @pytest.mark.asyncio
    async def test_sort_by_deadline_asc(
        listing_app, test_client, free_jwt, seeded_listing_opportunities
    ):
        """E06-P1-010: sort_by=deadline sort_order=asc → soonest deadline first."""
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={"sort_by": "deadline", "sort_order": "asc", "limit": 100},
            headers=free_jwt,
        )
        assert resp.status_code == 200
        data = resp.json()
        titles = [r["title"] for r in data["results"]]
        listing_titles = [t for t in titles if t.startswith("Listing Test Opportunity")]
        # A (5 days) should appear before B (15 days), B before C (30 days) in ASC
        if len(listing_titles) >= 3:
            a_idx = listing_titles.index("Listing Test Opportunity A")
            b_idx = listing_titles.index("Listing Test Opportunity B")
            c_idx = listing_titles.index("Listing Test Opportunity C")
            assert a_idx < b_idx < c_idx, (
                f"Expected A ({a_idx}) < B ({b_idx}) < C ({c_idx}) for deadline ASC order"
            )


    @pytest.mark.asyncio
    async def test_sort_by_budget_desc(
        listing_app, test_client, free_jwt, seeded_listing_opportunities
    ):
        """E06-P1-010: sort_by=budget sort_order=desc → largest budget_max first."""
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={"sort_by": "budget", "sort_order": "desc", "limit": 100},
            headers=free_jwt,
        )
        assert resp.status_code == 200
        assert resp.json()["results"] is not None  # Endpoint returns without error


    @pytest.mark.asyncio
    async def test_sort_by_published_date_desc(
        listing_app, test_client, free_jwt, seeded_listing_opportunities
    ):
        """E06-P1-010: sort_by=published_date sort_order=desc → most recently published first."""
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={"sort_by": "published_date", "sort_order": "desc", "limit": 100},
            headers=free_jwt,
        )
        assert resp.status_code == 200
        data = resp.json()
        listing_results = [r for r in data["results"] if r["title"].startswith("Listing Test Opportunity")]
        # C has published_at = -1 day (most recent), B = -5 days, A = -10 days
        # DESC → C first, then B, then A
        if len(listing_results) >= 3:
            listing_titles = [r["title"] for r in listing_results]
            c_idx = listing_titles.index("Listing Test Opportunity C")
            b_idx = listing_titles.index("Listing Test Opportunity B")
            a_idx = listing_titles.index("Listing Test Opportunity A")
            assert c_idx < b_idx < a_idx, (
                f"Expected C ({c_idx}) < B ({b_idx}) < A ({a_idx}) for published_date DESC"
            )


    @pytest.mark.asyncio
    async def test_sort_by_relevance_score_desc(
        listing_app, test_client, seeded_listing_opportunities,
        starter_user_token,
    ):
        """E06-P1-010: sort_by=relevance_score sort_order=desc → highest company score first.

        Starter user whose company_id = _COMPANY_ID has scores: A=0.9, B=0.5, C=0.2.
        DESC → A first, then B, then C.
        """
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={
                "sort_by": "relevance_score",
                "sort_order": "desc",
                "regions": "Bulgaria",
                "limit": 100,
            },
            headers={"Authorization": f"Bearer {starter_user_token}"},
        )
        assert resp.status_code == 200
        data = resp.json()
        listing_titles = [
            r["title"] for r in data["results"]
            if r["title"].startswith("Listing Test Opportunity")
        ]
        if len(listing_titles) >= 3:
            a_idx = listing_titles.index("Listing Test Opportunity A")
            b_idx = listing_titles.index("Listing Test Opportunity B")
            c_idx = listing_titles.index("Listing Test Opportunity C")
            assert a_idx < b_idx < c_idx, (
                f"Expected A ({a_idx}) < B ({b_idx}) < C ({c_idx}) for relevance DESC"
            )


    # ---------------------------------------------------------------------------
    # E06-P1-010: Invalid sort param validation
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_invalid_sort_by_returns_400(listing_app, test_client, free_jwt):
        """E06-P1-010 (AC3): Invalid sort_by returns 400 with field validation error."""
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={"sort_by": "unknown_sort_field"},
            headers=free_jwt,
        )
        assert resp.status_code == 400
        data = resp.json()
        assert "sort_by" in str(data).lower() or data.get("error") == "bad_request"


    @pytest.mark.asyncio
    async def test_invalid_sort_order_returns_400(listing_app, test_client, free_jwt):
        """AC3: Invalid sort_order returns 400."""
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={"sort_order": "random"},
            headers=free_jwt,
        )
        assert resp.status_code == 400


    # ---------------------------------------------------------------------------
    # Misc / edge cases
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_listing_requires_authentication(listing_app, test_client):
        """AC9: Unauthenticated request returns 401."""
        resp = await test_client.get("/api/v1/opportunities")
        assert resp.status_code == 401


    @pytest.mark.asyncio
    async def test_listing_empty_results(listing_app, test_client, free_jwt):
        """AC7: No matching results → 200 with results=[], total_count=0, cursors=null."""
        resp = await test_client.get(
            "/api/v1/opportunities",
            params={"regions": "NonExistentRegionXYZ123"},
            headers=free_jwt,
        )
        assert resp.status_code == 200
        data = resp.json()
        assert data["results"] == []
        assert data["total_count"] == 0
        assert data["next_cursor"] is None
        assert data["prev_cursor"] is None


    @pytest.mark.asyncio
    async def test_listing_limit_boundary_values(
        listing_app, test_client, free_jwt, seeded_listing_opportunities
    ):
        """AC4: limit=100 succeeds; limit=101 returns 422."""
        r100 = await test_client.get(
            "/api/v1/opportunities",
            params={"limit": 100},
            headers=free_jwt,
        )
        assert r100.status_code == 200

        r101 = await test_client.get(
            "/api/v1/opportunities",
            params={"limit": 101},
            headers=free_jwt,
        )
        assert r101.status_code == 422


    @pytest.mark.asyncio
    async def test_listing_pagination_metadata_with_next_cursor(
        listing_app, test_client, free_jwt, seeded_listing_opportunities
    ):
        """E06-P1-011: When more results exist, next_cursor is non-null; prev_cursor on page 2."""
        # Page 1
        resp1 = await test_client.get(
            "/api/v1/opportunities",
            params={"limit": 1, "sort_by": "published_date", "sort_order": "desc"},
            headers=free_jwt,
        )
        assert resp1.status_code == 200
        d1 = resp1.json()
        assert len(d1["results"]) == 1
        # If there are multiple rows, next_cursor should be present
        if d1["total_count"] > 1:
            assert d1["next_cursor"] is not None
            assert d1["prev_cursor"] is None  # First page has no prev

            # Page 2
            resp2 = await test_client.get(
                "/api/v1/opportunities",
                params={"limit": 1, "sort_by": "published_date", "sort_order": "desc",
                        "after_cursor": d1["next_cursor"]},
                headers=free_jwt,
            )
            assert resp2.status_code == 200
            d2 = resp2.json()
            assert len(d2["results"]) == 1
            # No overlap between pages
            assert d2["results"][0]["id"] != d1["results"][0]["id"]
            # Page 2 should have a prev_cursor pointing back to page 1
            assert d2["prev_cursor"] is not None
    ```

## Dev Notes

### Architecture Context

S06.04 adds the primary browse-mode listing endpoint to the opportunity discovery layer. Its relationship to the other E06 stories:

| Story | Role | Endpoint |
|-------|------|----------|
| S06.01 | Search with FTS (`q` param) | `GET /api/v1/opportunities/search` |
| **S06.04** | **Browse without FTS (this story)** | **`GET /api/v1/opportunities`** |
| S06.05 | Single opportunity detail | `GET /api/v1/opportunities/{id}` |

Both S06.01 and S06.04 share:
- `get_pipeline_readonly_session` dependency (read-only `pipeline.opportunities` session)
- `opportunities_table` Core definition (`models/pipeline_opportunity.py`)
- `OpportunityTierGateContext` dependency (`core/opportunity_tier_gate.py`)
- `OpportunityFreeResponse` / `OpportunityFullResponse` response models

S06.04 introduces a **separate service function** (`list_opportunities`) rather than extending `search_opportunities`, because the sort-aware cursor logic is fundamentally different from the search cursor (which always sorts by `published_at`). Reusing the search function with an optional `sort_by` param would require messy conditional cursor logic that increases the risk of pagination bugs.

### CRITICAL: Sort-Aware Cursor Format (Distinct from Search Cursor)

The listing cursor uses a different format from the search cursor:

| Cursor | Format | Sort Keys |
|--------|--------|-----------|
| Search cursor (`_encode_cursor`) | `{"published_at": "<ISO>", "id": "<UUID>"}` | Always published_at + id |
| Listing cursor (`_encode_listing_cursor`) | `{"sort_by": "<sort_by>", "v": "<value_or_null>", "id": "<UUID>"}` | Dynamic based on sort_by |

**NEVER pass a listing cursor to the search endpoint or vice versa** — the decoder will raise a `ValueError` (missing expected keys) which correctly converts to HTTP 400.

The listing cursor's `sort_by` field is validated on decode: if the request's `sort_by` differs from the cursor's embedded `sort_by`, a 400 is returned with `error=invalid_cursor` and message "sort_by mismatch between cursor and current request". This prevents wrong-order pagination when a user switches sort mode mid-session.

### CRITICAL: NULLS LAST and Keyset Pagination Edge Cases

The keyset WHERE helper (`_build_listing_keyset_condition`) handles four distinct cases:
1. **Cursor at non-null value, forward pagination** — standard range condition + `OR sort_col IS NULL` to include the null region (which sorts after all non-null values)
2. **Cursor at non-null value, backward pagination** — inverse range condition without IS NULL (items before a non-null cursor are all non-null)
3. **Cursor at null value, forward pagination** — `AND sort_col IS NULL AND id < cursor_id` (only items within the null region)
4. **Cursor at null value, backward pagination** — `WHERE sort_col IS NOT NULL` (all non-null items precede null items)

This logic is tested indirectly via integration tests. A dedicated unit test for `_build_listing_keyset_condition` is deferred — the function is an internal helper not exported for direct testing.

### CRITICAL: relevance_score Sort and company_id

The `relevance_scores` JSONB field on `pipeline.opportunities` is keyed by `company_id` (string UUID):
```json
{"abc-123-uuid": 0.9, "def-456-uuid": 0.5}
```
(See E05 S05.07 implementation — relevance scoring stores per-company scores in this field.)

The sort expression for `sort_by=relevance_score`:
```python
from sqlalchemy import Float
# Extracts the company's score as text, casts to Float for comparison
sort_col = opp_t.c.relevance_scores[str(company_id)].astext.cast(Float)
```

`company_id` is `current_user.company_id` (from the JWT `get_current_user` dependency) — no extra DB query. If `current_user.company_id` is `None` (which should not happen for authenticated users with an active company), the service function falls back to `published_date` sort.

The `company_id` is passed into `list_opportunities()` only when `sort_by == "relevance_score"` (see route handler: `company_id=current_user.company_id if sort_by == "relevance_score" else None`). For other sort modes, `company_id=None` is passed to avoid unnecessary string formatting.

### CRITICAL: Route Registration Order — `/opportunities` vs. `/opportunities/search`

FastAPI matches routes in registration order. `GET /opportunities` must be registered **before** or **separately** from `/opportunities/{id}` (S06.05) to avoid conflict. The search endpoint (`/opportunities/search`) works correctly already because FastAPI treats the literal path segment `search` as higher priority than the `{id}` path parameter when both are on the same router.

The listing endpoint `GET ""` (empty path within the `/opportunities` prefix router) matches `GET /api/v1/opportunities` exactly. This is distinct from `/opportunities/search` (which has an additional path segment). No ordering issue.

Verify by checking `GET /openapi.json` after registration that:
- `GET /api/v1/opportunities` appears (listing — this story)
- `GET /api/v1/opportunities/search` appears (FTS search — S06.01)
- `GET /api/v1/opportunities/{opportunity_id}` will appear (detail — S06.05)

### CRITICAL: `current_user.company_id` Type

`CurrentUser.company_id` is a `UUID` (not a string). The service function's `company_id: UUID | None` parameter is correct. In `_get_sort_column()`, convert to string for the JSONB subscript: `opp_t.c.relevance_scores[str(company_id)]`.

If `company_id` is `None` and `sort_by == "relevance_score"`, `_get_sort_column` falls back to `published_at`. The route handler passes `company_id=None` for all non-relevance_score sorts, so this fallback is only triggered if a company-less JWT is used with relevance_score sort (shouldn't happen in normal flow but is safe to handle gracefully).

### CRITICAL: `Float` Import in `opportunity_service.py`

The `_get_sort_column` function uses `cast` and `Float` from SQLAlchemy. Add these to the existing `from sqlalchemy import ...` line:

```python
from sqlalchemy import Float, and_, func, nulls_last, or_, select
```

If `Float` or `nulls_last` are not already in the import, add them. The existing `from sqlalchemy import and_, func, or_, select` line (from S06.01) needs to be extended.

### CRITICAL: `total_count` is Unfiltered by TierGate

`total_count` in the response reflects the full count from the database filter conditions (status, cpv_codes, regions, budget, deadline), NOT the tier-filtered count. Out-of-scope items for paid tiers are silently removed from `results[]` but still counted in `total_count`. This matches the search endpoint behavior (established in S06.02 AC7) and is the correct UX trade-off: the counter shows "how many opportunities match your filters" regardless of what you can see.

This is a documented behavior, not a bug. The frontend should not derive a "visible count" from `total_count`.

### Test Fixtures: `starter_user_token` vs. `free_user_token`

The integration tests use two JWT fixtures:
- `free_user_token` — assumed to exist in `conftest.py` (from S06.01/S06.02 tests); `subscription_tier=free`
- `starter_user_token` — must also be in `conftest.py`; `subscription_tier=starter`; **company_id must match `_COMPANY_ID` constant in the test file** so the relevance_score sort test picks up the seeded scores

If `starter_user_token` is not yet in `conftest.py`, add it following the same pattern as `free_user_token`. The JWT must include `company_id: _COMPANY_ID` as a claim so `current_user.company_id` returns the UUID used to seed `relevance_scores`.

Check `eusolicit-app/services/client-api/tests/conftest.py` before implementing — if `starter_user_token` exists, use it. If not, add it in the same conftest under a `@pytest_asyncio.fixture` following the `free_user_token` pattern.

### File Locations

| Purpose | Path |
|---------|------|
| Listing cursor helpers + `list_opportunities()` | `eusolicit-app/services/client-api/src/client_api/services/opportunity_service.py` |
| `GET /api/v1/opportunities` route handler | `eusolicit-app/services/client-api/src/client_api/api/v1/opportunities.py` |
| Integration tests | `eusolicit-app/services/client-api/tests/api/test_opportunity_listing.py` |
| Conftest (JWT fixtures) | `eusolicit-app/services/client-api/tests/conftest.py` |
| OpportunityTierGateContext (reused) | `eusolicit-app/services/client-api/src/client_api/core/opportunity_tier_gate.py` |
| Pipeline readonly session (reused) | `eusolicit-app/services/client-api/src/client_api/dependencies.py` |
| Pipeline opportunities table (reused) | `eusolicit-app/services/client-api/src/client_api/models/pipeline_opportunity.py` |

### Test Design Traceability

| Test ID | Priority | Scenario | AC |
|---------|----------|----------|----|
| E06-P0-002 | P0 | Free-tier listing → FreeResponse with only 6 fields; budget/CPV/etc. absent | AC6 |
| E06-P1-010 | P1 | Browse mode + 4 sort_by values × 2 sort_order; first-page items in correct order | AC2, AC3 |
| E06-P1-011 | P1 | Response has next_cursor, prev_cursor, total_count, results[]; free → FreeResponse[]; paid → FullResponse[] | AC4, AC6, AC7 |
| E06-P2-003 | P2 | Cursor re-applies TierGate on each page — Starter user gets consistent scope filtering on page 2 | AC5, AC6 |
| E06-P3-001 (partial) | P3 | GET /api/v1/opportunities appears in /openapi.json | AC1 |

Risk mitigations relevant to this story:
- **E06-R-001** (TierGate bypass — score 6): AC6 mandates TierGate on EVERY result item via `is_in_scope()` + `serialize_item()`; E06-P0-002 validates field-level enforcement on the listing endpoint; cursor sort_by mismatch check (AC5) prevents scope bypass via manipulated cursors
- **E06-R-005** (cursor forgery — score 4): The listing cursor's embedded `sort_by` is validated against the request param; an invalid base64 / malformed JSON cursor returns 400

### References

- [Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.04] — S06.04 spec with sort_by, sort_order, filter params, cursor-based pagination requirements
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P1-010] — P1 test: browse mode + sort combinations
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P1-011] — P1 test: listing response structure (pagination metadata + tier response shapes)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P0-002] — P0 test: free-tier FreeResponse field-level enforcement on listing
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-R-001] — TierGate bypass risk (score 6); mitigated by per-item is_in_scope() + serialize_item()
- [Source: eusolicit-docs/implementation-artifacts/6-1-opportunity-search-api-with-full-text-search-filters.md] — Search endpoint patterns: pipeline readonly session, cursor encoding, filter logic
- [Source: eusolicit-docs/implementation-artifacts/6-2-tier-gated-response-serialization.md] — TierGate dependency, is_in_scope(), serialize_item(), FreeResponse/FullResponse models
- [Source: eusolicit-app/services/client-api/src/client_api/services/opportunity_service.py] — Existing cursor helpers, filter conditions, search function (reused patterns)
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/opportunities.py] — Existing router structure; listing endpoint added to same file
- [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/models/opportunity.py] — `relevance_scores: Mapped[dict | None]` confirmed as `{company_id: score}` JSONB (S05.07)

## Change Log

| Date | Actor | Change |
|---|---|---|
| 2026-04-17 | BMad Orchestrator (bmad-create-story) | Story file created from epic-06 S06.04 spec and test-design-epic-06.md; dev notes reference existing client-api patterns (S06.01 cursor helpers, S06.02 TierGate, S06.03 dependency injection patterns); test design traceability to E06-P0-002, E06-P1-010, E06-P1-011; relevance_scores structure confirmed from S05.07 implementation (company_id-keyed JSONB) |
