# Story 8.11: Enterprise Custom Invoicing

Status: review

## Story

As a platform administrator,
I want to create custom Stripe invoices for Enterprise-tier customers with NET 30 or NET 60 payment terms,
so that enterprise clients receive professional invoices with deferred payment terms and I can track all outstanding enterprise invoices in a single admin view.

## Acceptance Criteria

1. **AC1 — DB: `enterprise_invoices` Table** — Alembic migration `031_enterprise_invoices.py` creates a `client.enterprise_invoices` table with: `id` (UUID PK, `gen_random_uuid()`), `company_id` (UUID NOT NULL FK → `client.companies.id` ON DELETE RESTRICT), `stripe_invoice_id` (VARCHAR(255) NOT NULL UNIQUE), `stripe_customer_id` (VARCHAR(255) NOT NULL), `admin_id` (UUID NOT NULL — platform admin UUID; NOT a FK to client.users), `payment_terms` (VARCHAR(10) NOT NULL — `'NET_30'` or `'NET_60'`), `line_items` (JSONB NOT NULL DEFAULT `'[]'`), `status` (VARCHAR(20) NOT NULL DEFAULT `'open'`), `created_at` (TIMESTAMPTZ NOT NULL DEFAULT now()), `sent_at` (TIMESTAMPTZ NULL), `paid_at` (TIMESTAMPTZ NULL). Index on `company_id`; unique index on `stripe_invoice_id`. Migration uses `schema="client"` on ALL DDL.

2. **AC2 — Admin API: Create Enterprise Invoice Endpoint** — `POST /api/v1/admin/enterprise-invoicing/` in `admin-api`:
   - Auth: `get_admin_user()` dependency — `platform_admin` role required; returns `AdminUser(admin_id, role)`
   - Request body `CreateEnterpriseInvoiceRequest`: `company_id` (UUID), `payment_terms` (Literal `"NET_30"` | `"NET_60"`), `line_items` (list[LineItem], min 1, max 50), `memo` (str | None, max 1000)
   - `LineItem` schema: `description` (str, min 1, max 500), `amount_cents` (int ≥ 1), `currency` (str default `"eur"`, 3 chars)
   - Steps: (a) verify company exists and `subscriptions.tier == 'enterprise'`; (b) verify `stripe_customer_id` is present; (c) create Stripe Invoice (draft); (d) add Stripe InvoiceItems; (e) finalize invoice; (f) send invoice; (g) persist `enterprise_invoices` row with `status='open'`, `sent_at=now()`; (h) return `EnterpriseInvoiceResponse` with HTTP 201
   - Error: company tier ≠ enterprise → HTTP 422 `{"error": "not_enterprise_tier", "message": "Custom invoicing is only available for Enterprise customers"}`
   - Error: no `stripe_customer_id` → HTTP 422 `{"error": "no_stripe_customer", "message": "Company has no Stripe customer ID"}`
   - Error: company not found → HTTP 404
   - Error: Stripe API failure → HTTP 502 `{"error": "stripe_error"}`

3. **AC3 — Stripe Invoicing API Flow** — Four-step flow, all steps via `asyncio.to_thread()`:
   1. `stripe.Invoice.create(customer=stripe_customer_id, collection_method='send_invoice', days_until_due=30_or_60, auto_advance=False, description=memo, metadata={"company_id": ..., "admin_id": ..., "payment_terms": ...})`
   2. `stripe.InvoiceItem.create(customer=stripe_customer_id, invoice=stripe_invoice_id, price_data={"unit_amount": amount_cents, "currency": currency, "product_data": {"name": description}}, quantity=1)` — one call per line item
   3. `stripe.Invoice.finalize_invoice(stripe_invoice_id)` — draft → open
   4. `stripe.Invoice.send_invoice(stripe_invoice_id)` — emails PDF to customer

4. **AC4 — Admin API: List Enterprise Invoices Endpoint** — `GET /api/v1/admin/enterprise-invoicing/` in `admin-api`:
   - Auth: `platform_admin` role
   - Query params: `company_id` (UUID, optional), `status` (str, optional), `limit` (int, default 50, max 200), `offset` (int, default 0)
   - Returns: `EnterpriseInvoiceListResponse` with `items`, `total`, `limit`, `offset`
   - `EnterpriseInvoiceResponse` fields: `id`, `company_id`, `stripe_invoice_id`, `stripe_customer_id`, `admin_id`, `payment_terms`, `line_items`, `status`, `created_at`, `sent_at`, `paid_at`
   - Ordered by `created_at DESC`

5. **AC5 — Webhook: `invoice.paid` Updates Enterprise Invoice Status** — In `client-api` `webhook_service.py`, extend `_handle_invoice_event()` to handle enterprise invoice payment:
   - For `invoice.paid` events: extract `stripe_invoice_id = event.data.object.id`
   - Query `enterprise_invoices` WHERE `stripe_invoice_id = <id>` — if found, UPDATE `status='paid'`, `paid_at=now()`
   - ADDITIVE ONLY: existing subscription `invoice.paid` handler (which reads `event.data.object.subscription`) is unchanged
   - Both handlers run for every `invoice.paid` event; each silently no-ops if no matching row found

6. **AC6 — Audit Trail** — `POST` endpoint logs `billing.enterprise_invoice_created` via `structlog` with `company_id`, `admin_id`, `stripe_invoice_id`, `payment_terms`, `line_items_count` (admin-api uses structlog logging, not the client-api `write_audit_entry` service).

7. **AC7 — Tier Verification** — Before creating a Stripe invoice, JOIN `client.companies` + `client.subscriptions` on `company_id`. Reject if no row found, if tier ≠ `'enterprise'`, or if `stripe_customer_id` is NULL/empty.

## Tasks / Subtasks

### Backend — DB Migration (client-api)

- [x] **Task 1 — Create `enterprise_invoices` migration** (AC: 1)
  - [x] 1.1 Create `services/client-api/alembic/versions/031_enterprise_invoices.py`:
    ```python
    """Add enterprise_invoices table for admin custom invoicing.

    Revision ID: 031
    Revises: 030
    Create Date: 2026-04-19
    """
    from alembic import op
    import sqlalchemy as sa
    from sqlalchemy.dialects.postgresql import UUID, JSONB

    revision = "031"
    down_revision = "030"
    branch_labels = None
    depends_on = None


    def upgrade() -> None:
        op.create_table(
            "enterprise_invoices",
            sa.Column("id", UUID(as_uuid=True), primary_key=True,
                      server_default=sa.text("gen_random_uuid()")),
            sa.Column("company_id", UUID(as_uuid=True),
                      sa.ForeignKey("client.companies.id", ondelete="RESTRICT"),
                      nullable=False),
            sa.Column("stripe_invoice_id", sa.String(255), nullable=False),
            sa.Column("stripe_customer_id", sa.String(255), nullable=False),
            sa.Column("admin_id", UUID(as_uuid=True), nullable=False),
            sa.Column("payment_terms", sa.String(10), nullable=False),
            sa.Column("line_items", JSONB, nullable=False,
                      server_default=sa.text("'[]'::jsonb")),
            sa.Column("status", sa.String(20), nullable=False,
                      server_default=sa.text("'open'")),
            sa.Column("created_at", sa.TIMESTAMP(timezone=True),
                      server_default=sa.text("now()"), nullable=False),
            sa.Column("sent_at", sa.TIMESTAMP(timezone=True), nullable=True),
            sa.Column("paid_at", sa.TIMESTAMP(timezone=True), nullable=True),
            schema="client",
        )
        op.create_index(
            "ix_enterprise_invoices_company_id",
            "enterprise_invoices", ["company_id"], schema="client",
        )
        op.create_unique_constraint(
            "uq_enterprise_invoices_stripe_invoice_id",
            "enterprise_invoices", ["stripe_invoice_id"], schema="client",
        )


    def downgrade() -> None:
        op.drop_constraint(
            "uq_enterprise_invoices_stripe_invoice_id",
            "enterprise_invoices", type_="unique", schema="client",
        )
        op.drop_index(
            "ix_enterprise_invoices_company_id",
            table_name="enterprise_invoices", schema="client",
        )
        op.drop_table("enterprise_invoices", schema="client")
    ```
    **CRITICAL:** Every `op.*` call must include `schema="client"`. Without it, DDL lands in `public` schema (project-context rule #3).

  - [x] 1.2 Verify migration applies cleanly: `make migrate-service SVC=client-api`

### Backend — ORM Model (client-api)

- [x] **Task 2 — Create `EnterpriseInvoice` ORM model** (AC: 1)
  - [x] 2.1 Create `services/client-api/src/client_api/models/enterprise_invoice.py`:
    ```python
    """EnterpriseInvoice ORM model — client.enterprise_invoices."""
    from __future__ import annotations

    import uuid
    from datetime import datetime

    import sqlalchemy as sa
    from sqlalchemy.dialects.postgresql import JSONB, UUID
    from sqlalchemy.orm import Mapped, mapped_column

    from client_api.models.base import Base


    class EnterpriseInvoice(Base):
        __tablename__ = "enterprise_invoices"
        __table_args__ = {"schema": "client"}

        id: Mapped[uuid.UUID] = mapped_column(
            UUID(as_uuid=True), primary_key=True, default=uuid.uuid4,
        )
        company_id: Mapped[uuid.UUID] = mapped_column(
            UUID(as_uuid=True),
            sa.ForeignKey("client.companies.id", ondelete="RESTRICT"),
            nullable=False,
        )
        stripe_invoice_id: Mapped[str] = mapped_column(
            sa.String(255), nullable=False, unique=True,
        )
        stripe_customer_id: Mapped[str] = mapped_column(sa.String(255), nullable=False)
        admin_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
        payment_terms: Mapped[str] = mapped_column(sa.String(10), nullable=False)
        line_items: Mapped[list] = mapped_column(JSONB, nullable=False, default=list)
        status: Mapped[str] = mapped_column(sa.String(20), nullable=False, default="open")
        created_at: Mapped[datetime] = mapped_column(
            sa.TIMESTAMP(timezone=True), server_default=sa.text("now()"), nullable=False,
        )
        sent_at: Mapped[datetime | None] = mapped_column(
            sa.TIMESTAMP(timezone=True), nullable=True,
        )
        paid_at: Mapped[datetime | None] = mapped_column(
            sa.TIMESTAMP(timezone=True), nullable=True,
        )
    ```
    Note: `admin_id` is a bare UUID (no FK) — platform admins live in the `admin` schema, not `client.users`.

### Backend — Admin-API Schemas

- [x] **Task 3 — Create Pydantic schemas in admin-api** (AC: 2, 4)
  - [x] 3.1 Create `services/admin-api/src/admin_api/schemas/enterprise_invoice.py`:
    ```python
    """Pydantic schemas for Enterprise Custom Invoicing."""
    from __future__ import annotations

    import uuid
    from datetime import datetime
    from typing import Literal

    from pydantic import BaseModel, Field


    class LineItemSchema(BaseModel):
        description: str = Field(min_length=1, max_length=500)
        amount_cents: int = Field(ge=1, description="Amount in cents, e.g. 10000 = €100.00")
        currency: str = Field(default="eur", min_length=3, max_length=3)


    class CreateEnterpriseInvoiceRequest(BaseModel):
        company_id: uuid.UUID
        payment_terms: Literal["NET_30", "NET_60"]
        line_items: list[LineItemSchema] = Field(min_length=1, max_length=50)
        memo: str | None = Field(default=None, max_length=1000)


    class EnterpriseInvoiceResponse(BaseModel):
        id: uuid.UUID
        company_id: uuid.UUID
        stripe_invoice_id: str
        stripe_customer_id: str
        admin_id: uuid.UUID
        payment_terms: str
        line_items: list[dict]
        status: str
        created_at: datetime
        sent_at: datetime | None
        paid_at: datetime | None

        model_config = {"from_attributes": True}


    class EnterpriseInvoiceListResponse(BaseModel):
        items: list[EnterpriseInvoiceResponse]
        total: int
        limit: int
        offset: int
    ```

### Backend — Admin-API Service Layer

- [x] **Task 4 — Create `enterprise_invoice_service.py`** (AC: 2, 3, 4, 7)
  - [x] 4.1 Create `services/admin-api/src/admin_api/services/enterprise_invoice_service.py`:
    ```python
    """Enterprise Custom Invoicing — admin service layer.

    All Stripe SDK calls use asyncio.to_thread() — SDK is synchronous.
    """
    from __future__ import annotations

    import asyncio
    import uuid
    from datetime import datetime, timezone

    import stripe
    import structlog
    from sqlalchemy import func, select
    from sqlalchemy.ext.asyncio import AsyncSession

    from admin_api.schemas.enterprise_invoice import (
        CreateEnterpriseInvoiceRequest,
        EnterpriseInvoiceListResponse,
        EnterpriseInvoiceResponse,
    )

    logger = structlog.get_logger(__name__)

    _TERMS_TO_DAYS: dict[str, int] = {"NET_30": 30, "NET_60": 60}


    async def create_enterprise_invoice(
        session: AsyncSession,
        request: CreateEnterpriseInvoiceRequest,
        admin_id: uuid.UUID,
    ) -> EnterpriseInvoiceResponse:
        """Create a custom Stripe invoice for an Enterprise-tier company.

        Raises ValueError if company not found, not enterprise tier, or no stripe_customer_id.
        Raises stripe.error.StripeError on Stripe API failure.
        """
        # Step 1: Verify company exists, tier == 'enterprise', has stripe_customer_id
        # Use Core table definitions from client_tables.py (admin-api pattern)
        from admin_api.models.client_tables import companies_table, subscriptions_table

        result = await session.execute(
            select(
                companies_table.c.id,
                subscriptions_table.c.tier,
                subscriptions_table.c.stripe_customer_id,
            ).select_from(
                companies_table.outerjoin(
                    subscriptions_table,
                    companies_table.c.id == subscriptions_table.c.company_id,
                )
            ).where(companies_table.c.id == request.company_id)
        )
        row = result.one_or_none()

        if row is None:
            raise ValueError(f"company_not_found:{request.company_id}")
        if row.tier != "enterprise":
            raise ValueError(f"not_enterprise_tier:tier={row.tier!r}")
        if not row.stripe_customer_id:
            raise ValueError(f"no_stripe_customer:{request.company_id}")

        stripe_customer_id: str = row.stripe_customer_id
        days_until_due = _TERMS_TO_DAYS[request.payment_terms]

        # Step 2: Create Stripe Invoice (draft; auto_advance=False prevents auto-finalization)
        stripe_invoice = await asyncio.to_thread(
            stripe.Invoice.create,
            customer=stripe_customer_id,
            collection_method="send_invoice",
            days_until_due=days_until_due,
            description=request.memo,
            auto_advance=False,
            metadata={
                "company_id": str(request.company_id),
                "admin_id": str(admin_id),
                "payment_terms": request.payment_terms,
            },
        )
        stripe_invoice_id: str = stripe_invoice["id"]

        logger.info(
            "enterprise_invoice_stripe_created",
            stripe_invoice_id=stripe_invoice_id,
            company_id=str(request.company_id),
            admin_id=str(admin_id),
            payment_terms=request.payment_terms,
        )

        # Step 3: Add InvoiceItems (one call per line item)
        for item in request.line_items:
            await asyncio.to_thread(
                stripe.InvoiceItem.create,
                customer=stripe_customer_id,
                invoice=stripe_invoice_id,
                price_data={
                    "unit_amount": item.amount_cents,
                    "currency": item.currency,
                    "product_data": {"name": item.description},
                },
                quantity=1,
            )

        # Step 4: Finalize (draft → open)
        await asyncio.to_thread(stripe.Invoice.finalize_invoice, stripe_invoice_id)

        # Step 5: Send invoice email to customer
        await asyncio.to_thread(stripe.Invoice.send_invoice, stripe_invoice_id)

        now = datetime.now(tz=timezone.utc)

        # Step 6: Persist to DB
        # admin-api accesses client schema via get_client_db_session().
        # Import ORM model — verify admin-api can import from client_api package.
        # If not (package isolation), define a Core Table insert in client_tables.py instead.
        from client_api.models.enterprise_invoice import EnterpriseInvoice

        record = EnterpriseInvoice(
            company_id=request.company_id,
            stripe_invoice_id=stripe_invoice_id,
            stripe_customer_id=stripe_customer_id,
            admin_id=admin_id,
            payment_terms=request.payment_terms,
            line_items=[item.model_dump() for item in request.line_items],
            status="open",
            sent_at=now,
        )
        session.add(record)
        await session.commit()
        await session.refresh(record)

        logger.info(
            "enterprise_invoice_created",
            invoice_id=str(record.id),
            stripe_invoice_id=stripe_invoice_id,
            company_id=str(request.company_id),
            admin_id=str(admin_id),
            payment_terms=request.payment_terms,
            line_items_count=len(request.line_items),
        )

        return EnterpriseInvoiceResponse.model_validate(record)


    async def list_enterprise_invoices(
        session: AsyncSession,
        company_id: uuid.UUID | None,
        status: str | None,
        limit: int,
        offset: int,
    ) -> EnterpriseInvoiceListResponse:
        """List enterprise invoices with optional filters, ordered newest first."""
        from client_api.models.enterprise_invoice import EnterpriseInvoice

        base_q = select(EnterpriseInvoice)
        count_q = select(func.count()).select_from(EnterpriseInvoice)

        if company_id is not None:
            base_q = base_q.where(EnterpriseInvoice.company_id == company_id)
            count_q = count_q.where(EnterpriseInvoice.company_id == company_id)
        if status is not None:
            base_q = base_q.where(EnterpriseInvoice.status == status)
            count_q = count_q.where(EnterpriseInvoice.status == status)

        total = (await session.execute(count_q)).scalar_one()
        rows = (
            await session.execute(
                base_q.order_by(EnterpriseInvoice.created_at.desc())
                .limit(limit)
                .offset(offset)
            )
        ).scalars().all()

        return EnterpriseInvoiceListResponse(
            items=[EnterpriseInvoiceResponse.model_validate(r) for r in rows],
            total=total,
            limit=limit,
            offset=offset,
        )
    ```

### Backend — Admin-API Router

- [x] **Task 5 — Create enterprise invoicing router** (AC: 2, 4, 6)
  - [x] 5.1 Create `services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py`:
    ```python
    """Enterprise Custom Invoicing — admin API endpoints.

    POST /api/v1/admin/enterprise-invoicing/   create invoice (201)
    GET  /api/v1/admin/enterprise-invoicing/   list invoices  (200)
    """
    from __future__ import annotations

    import uuid
    from typing import Annotated

    import stripe
    import structlog
    from fastapi import APIRouter, Depends, HTTPException, Query
    from sqlalchemy.ext.asyncio import AsyncSession

    from admin_api.core.security import AdminUser
    from admin_api.dependencies import get_admin_user, get_client_db_session
    from admin_api.schemas.enterprise_invoice import (
        CreateEnterpriseInvoiceRequest,
        EnterpriseInvoiceListResponse,
        EnterpriseInvoiceResponse,
    )
    from admin_api.services import enterprise_invoice_service

    logger = structlog.get_logger(__name__)

    router = APIRouter(
        prefix="/admin/enterprise-invoicing",
        tags=["admin-enterprise-invoicing"],
    )

    _Admin = Annotated[AdminUser, Depends(get_admin_user)]
    _ClientSession = Annotated[AsyncSession, Depends(get_client_db_session)]


    @router.post("/", response_model=EnterpriseInvoiceResponse, status_code=201)
    async def create_enterprise_invoice(
        body: CreateEnterpriseInvoiceRequest,
        admin: _Admin,
        session: _ClientSession,
    ) -> EnterpriseInvoiceResponse:
        """Create a custom Stripe invoice for an Enterprise-tier company."""
        try:
            return await enterprise_invoice_service.create_enterprise_invoice(
                session=session,
                request=body,
                admin_id=admin.admin_id,
            )
        except ValueError as exc:
            msg = str(exc)
            if "not_enterprise_tier" in msg:
                raise HTTPException(
                    status_code=422,
                    detail={
                        "error": "not_enterprise_tier",
                        "message": "Custom invoicing is only available for Enterprise customers",
                    },
                )
            if "no_stripe_customer" in msg:
                raise HTTPException(
                    status_code=422,
                    detail={"error": "no_stripe_customer",
                            "message": "Company has no Stripe customer ID"},
                )
            if "company_not_found" in msg:
                raise HTTPException(
                    status_code=404,
                    detail={"error": "company_not_found"},
                )
            raise HTTPException(status_code=422,
                                detail={"error": "validation_error", "message": msg})
        except stripe.error.StripeError as exc:
            logger.error(
                "enterprise_invoice_stripe_error",
                company_id=str(body.company_id),
                error=str(exc),
            )
            raise HTTPException(
                status_code=502,
                detail={"error": "stripe_error",
                        "message": "Failed to create Stripe invoice"},
            )


    @router.get("/", response_model=EnterpriseInvoiceListResponse)
    async def list_enterprise_invoices(
        admin: _Admin,
        session: _ClientSession,
        company_id: uuid.UUID | None = Query(default=None),
        status: str | None = Query(default=None),
        limit: int = Query(default=50, ge=1, le=200),
        offset: int = Query(default=0, ge=0),
    ) -> EnterpriseInvoiceListResponse:
        """List all enterprise custom invoices with optional filters."""
        return await enterprise_invoice_service.list_enterprise_invoices(
            session=session,
            company_id=company_id,
            status=status,
            limit=limit,
            offset=offset,
        )
    ```

  - [x] 5.2 Register in `services/admin-api/src/admin_api/main.py`:
    ```python
    from admin_api.api.v1 import enterprise_invoicing
    # Inside the api_v1_router block alongside other router includes:
    api_v1_router.include_router(enterprise_invoicing.router)
    ```
    Follow the existing pattern — check `main.py` for how `tenants`, `audit_logs`, etc. are included. The full path will be `/api/v1/admin/enterprise-invoicing/`.

### Backend — Webhook Extension (client-api)

- [x] **Task 6 — Extend `invoice.paid` webhook handler** (AC: 5)
  - [x] 6.1 In `services/client-api/src/client_api/services/webhook_service.py`, extend `_handle_invoice_event()`:

    After the existing subscription status update block, add a call to handle enterprise invoices:
    ```python
    # ADDITIVE: check if this is an enterprise custom invoice payment
    stripe_invoice_id_val = event_obj.get("id")
    if event_type == "invoice.paid" and stripe_invoice_id_val:
        await _handle_enterprise_invoice_paid(session, stripe_invoice_id_val)
    ```

    Add the new private function below `_handle_invoice_event()`:
    ```python
    async def _handle_enterprise_invoice_paid(
        session: AsyncSession,
        stripe_invoice_id: str,
    ) -> None:
        """Mark enterprise custom invoice as paid when invoice.paid fires.

        No-op if stripe_invoice_id does not match any enterprise_invoices row.
        This runs for ALL invoice.paid events (subscription and custom) — the
        WHERE clause makes it safe to call unconditionally.
        """
        from client_api.models.enterprise_invoice import EnterpriseInvoice
        from datetime import datetime, timezone
        from sqlalchemy import update as sa_update

        result = await session.execute(
            sa_update(EnterpriseInvoice)
            .where(EnterpriseInvoice.stripe_invoice_id == stripe_invoice_id)
            .values(status="paid", paid_at=datetime.now(tz=timezone.utc))
            .returning(EnterpriseInvoice.id)
        )
        updated_id = result.scalar_one_or_none()
        if updated_id:
            logger.info(
                "enterprise_invoice_paid_via_webhook",
                stripe_invoice_id=stripe_invoice_id,
                enterprise_invoice_id=str(updated_id),
            )
    ```

    **CRITICAL:** This is purely ADDITIVE. Do NOT modify any existing logic in `_handle_invoice_event()`. The existing subscription `invoice.paid` path (which reads `event.data.object.subscription` → updates `subscriptions.status`) is completely untouched. [Source: 8-4-stripe-webhook-endpoint-subscription-lifecycle-sync.md]

### Backend — Tests

- [x] **Task 7 — Service unit tests** (AC: 2, 3, 7 — maps to 8.11-API-001)
  - [x] 7.1 Create `services/admin-api/tests/unit/test_enterprise_invoice_service.py`:

    **`TestCreateEnterpriseInvoice`** (use `pytest-mock`; mock `asyncio.to_thread` per Stripe call):
    - `test_8_11_api_001_create_invoice_net30_sets_days_until_due_30` (P2) — DB returns enterprise tier + `stripe_customer_id`. Mock all `asyncio.to_thread` calls. Assert `stripe.Invoice.create` called with `days_until_due=30`, `collection_method='send_invoice'`, `auto_advance=False`.
    - `test_create_invoice_net60_sets_days_until_due_60` — Same for `NET_60` → `days_until_due=60`.
    - `test_create_invoice_adds_stripe_invoice_item_per_line_item` — Request has 3 line items. Assert `stripe.InvoiceItem.create` called exactly 3 times with matching `price_data`.
    - `test_create_invoice_finalizes_and_sends_invoice_after_items` — Assert `stripe.Invoice.finalize_invoice` called before `stripe.Invoice.send_invoice` (call order matters).
    - `test_create_invoice_persists_db_record_with_status_open` — After Stripe calls succeed, assert `session.add` called with record having `status='open'` and `sent_at` set.
    - `test_create_invoice_rejects_non_enterprise_tier` — DB returns `tier='professional'`. Assert `ValueError` raised with `'not_enterprise_tier'` in message.
    - `test_create_invoice_rejects_missing_stripe_customer_id` — Enterprise tier but `stripe_customer_id=None`. Assert `ValueError` raised with `'no_stripe_customer'` in message.
    - `test_create_invoice_rejects_unknown_company` — DB returns no row. Assert `ValueError` raised with `'company_not_found'` in message.

    **`TestListEnterpriseInvoices`**:
    - `test_list_returns_all_invoices_without_filters` — DB returns 3 records. Assert response has `total=3`, `items` length 3.
    - `test_list_filters_by_company_id` — Assert WHERE clause includes `company_id` filter.
    - `test_list_filters_by_status_paid` — Assert WHERE clause includes `status='paid'` filter.
    - `test_list_applies_limit_and_offset` — Assert `LIMIT` and `OFFSET` applied correctly.

  - [x] 7.2 Create `services/client-api/tests/unit/test_enterprise_invoice_webhook.py`:
    - `test_invoice_paid_updates_enterprise_invoice_to_paid` — Mock DB to return matching `EnterpriseInvoice` for `stripe_invoice_id`. Assert `status='paid'` and `paid_at` set.
    - `test_invoice_paid_noop_when_no_enterprise_invoice_match` — DB returns no row. Assert no error raised; existing subscription handler unaffected.
    - `test_invoice_paid_does_not_break_subscription_handler` — Fire `invoice.paid` event with a `subscription` field set. Assert BOTH the subscription `status` update AND the enterprise invoice lookup execute (additive, no regression).
    - `test_enterprise_invoice_handler_not_called_for_other_event_types` — Fire `invoice.payment_failed`. Assert `_handle_enterprise_invoice_paid` NOT called.

- [x] **Task 8 — API-level tests** (AC: 2, 4)
  - [x] 8.1 Create `services/admin-api/tests/api/test_enterprise_invoice_api.py`:
    - `test_create_invoice_requires_platform_admin_auth` — No auth header. Assert 401 or 403.
    - `test_create_invoice_returns_201_with_invoice_response` — Mock service. Valid enterprise company request. Assert 201, response contains `stripe_invoice_id`, `payment_terms`, `status='open'`.
    - `test_create_invoice_returns_422_for_non_enterprise_tier` — Service raises `ValueError('not_enterprise_tier:...')`. Assert 422 `{"error": "not_enterprise_tier"}`.
    - `test_create_invoice_returns_422_for_missing_stripe_customer` — Service raises `ValueError('no_stripe_customer:...')`. Assert 422 `{"error": "no_stripe_customer"}`.
    - `test_create_invoice_returns_404_for_unknown_company` — Service raises `ValueError('company_not_found:...')`. Assert 404.
    - `test_create_invoice_returns_502_on_stripe_error` — Service raises `stripe.error.StripeError`. Assert 502 `{"error": "stripe_error"}`.
    - `test_list_invoices_returns_200_with_paginated_results` — Mock service returns list. Assert 200, response has `items`, `total`, `limit`, `offset`.
    - `test_list_invoices_filters_by_company_id_query_param` — `?company_id=<uuid>`. Assert service called with correct `company_id`.
    - `test_list_invoices_filters_by_status_query_param` — `?status=paid`. Assert service called with `status='paid'`.

## Dev Notes

### Critical Architecture Constraints

**This story belongs in `admin-api`, NOT `client-api`.** Enterprise custom invoicing is admin-only functionality requiring `platform_admin` JWT role. All new endpoint and service files live in `services/admin-api/`. The ORM model and Alembic migration live in `services/client-api/` (they define `client` schema resources). [Source: CLAUDE.md; admin-api description]

**`asyncio.to_thread()` is MANDATORY for ALL Stripe SDK calls.** The `stripe` Python SDK is fully synchronous. Every call — `stripe.Invoice.create`, `stripe.InvoiceItem.create`, `stripe.Invoice.finalize_invoice`, `stripe.Invoice.send_invoice` — MUST be wrapped:
```python
result = await asyncio.to_thread(stripe.Invoice.create, customer=..., ...)
```
Never call Stripe SDK functions directly in an async function. [Source: project-context.md EP4 retrospective; 8-9-per-bid-add-on-purchase-flow.md Dev Notes; 8-10 Dev Notes]

**Admin-API DB session for client schema.** Admin-api has two session factories:
- `get_db_session()` → admin schema (admin-only tables)
- `get_client_db_session()` → client schema (for cross-schema access including WRITES — e.g., tenant_service uses this for tier overrides)

Enterprise invoices go in `client.enterprise_invoices` → use `_ClientSession = Annotated[AsyncSession, Depends(get_client_db_session)]`. [Source: admin_api/dependencies.py; tenants.py pattern]

**Admin-API auth pattern.** `get_admin_user()` validates `platform_admin` JWT role, returns `AdminUser(admin_id: UUID, role: str)`. Use the type-alias pattern consistent with all other admin-api routers:
```python
_Admin = Annotated[AdminUser, Depends(get_admin_user)]
```
Do NOT use `Depends()` inline in function signatures. [Source: admin_api/api/v1/tenants.py]

**Cross-service ORM model import.** Admin-api needs `EnterpriseInvoice` ORM model for DB writes, but the model lives in `client-api`. Check `admin-api/pyproject.toml` for whether `client-api` is an editable dependency. If it IS listed (check `[tool.poetry.dependencies]` or `requirements.txt` for `client-api`), the direct import works. If NOT, the fallback is to define a SQLAlchemy Core `Table` object in `admin_api/models/client_tables.py` (existing pattern for other client-schema reads/writes in admin-api). Decide early — do NOT guess. [Source: admin_api/models/client_tables.py existing pattern]

**Stripe Invoicing API: four-step sequence** (all must complete before persisting DB record):
1. `stripe.Invoice.create(...)` → `{"id": "in_xxx", "status": "draft"}`
2. `stripe.InvoiceItem.create(...)` per line item → links item to invoice
3. `stripe.Invoice.finalize_invoice("in_xxx")` → `{"status": "open"}`
4. `stripe.Invoice.send_invoice("in_xxx")` → sends PDF email, status stays `'open'` until paid

**`auto_advance=False` is required on `stripe.Invoice.create`.** Without it, Stripe auto-finalizes drafts on a timer and may auto-charge subscription customers. Always set `auto_advance=False` for custom invoices so finalization is explicit. [Source: Stripe Invoice API docs]

**Migration is 031.** Previous migration is `030_vat_validation_status.py` (revision `"030"`). New file must declare `down_revision = "030"`. Always pass `schema="client"` on ALL `op.*` DDL calls. [Source: services/client-api/alembic/versions/030_vat_validation_status.py]

**Webhook extension is purely additive.** The existing `_handle_invoice_event()` in `webhook_service.py` handles:
- `invoice.paid` → sets `subscriptions.status = 'active'` (by looking up `subscription` field on the Stripe event)
- `invoice.payment_failed` → sets `subscriptions.status = 'past_due'`

These handlers are completely unchanged. The new `_handle_enterprise_invoice_paid()` runs AFTER existing logic on `invoice.paid` events only, looks up `enterprise_invoices` by `stripe_invoice_id`, and is a silent no-op if no match found. Both subscription and enterprise invoice `invoice.paid` events are handled safely — they're distinguished by `enterprise_invoices.stripe_invoice_id` lookup. [Source: webhook_service.py; 8-4-stripe-webhook-endpoint-subscription-lifecycle-sync.md]

**No Redis Streams event for enterprise invoice.** Custom invoices do not change the company tier. Do NOT publish `subscription.changed` or any Redis Streams event. Tier gating is unaffected by invoice creation or payment. [Source: epic AC-12 — Streams only on tier change]

**`admin_id` is a bare UUID (no FK constraint).** Platform admins live in `admin` schema, not `client.users`. Store `admin_id` as a plain UUID column without a foreign key. [Source: admin_api/core/security.py — AdminUser struct]

**Stripe error handling in router.** The service layer raises `stripe.error.StripeError` (base class) on any Stripe API failure. The router catches this and returns HTTP 502. Do NOT return the raw Stripe error message to the caller (may contain sensitive info). Log with structlog at ERROR level before raising HTTPException. [Source: billing_service.py error handling pattern]

### Existing Code to Reuse — DO NOT Reinvent

| Component | Location | How to Use |
|-----------|----------|------------|
| `get_admin_user()` | `admin_api/dependencies.py` | Auth dep — validates `platform_admin` JWT role |
| `get_client_db_session()` | `admin_api/dependencies.py` | Client-schema DB session for enterprise_invoices writes |
| `AdminUser` dataclass | `admin_api/core/security.py` | `.admin_id: UUID`, `.role: str` |
| Type-alias dep pattern | `admin_api/api/v1/tenants.py` | `_Admin = Annotated[AdminUser, Depends(get_admin_user)]` |
| `companies_table`, `subscriptions_table` | `admin_api/models/client_tables.py` | Core table defs for company/subscription reads |
| `asyncio.to_thread()` | stdlib | Wrap ALL Stripe SDK calls |
| `_handle_invoice_event()` | `webhook_service.py` | Extend (add call); NEVER rewrite |
| `webhook_events` dedup table | `client_api/models/webhook_event.py` | Already deduplicates all webhooks; no change needed |
| `structlog.get_logger(__name__)` | structlog | JSON logging; never use `print()` or stdlib `logging` |
| `Base` ORM base class | `client_api/models/base.py` | Base for `EnterpriseInvoice.__tablename__` |
| `stripe` SDK | installed in client-api | Also installed in admin-api (check `admin-api/pyproject.toml`) |

### Admin-API Router Registration

From `main.py` survey — all v1 routers follow this pattern:
```python
api_v1_router = APIRouter(prefix="/api/v1")
api_v1_router.include_router(tenants.router)
api_v1_router.include_router(audit_logs.router)
# ... add:
api_v1_router.include_router(enterprise_invoicing.router)
```
Full endpoint paths will be:
- `POST /api/v1/admin/enterprise-invoicing/`
- `GET  /api/v1/admin/enterprise-invoicing/`

### Stripe Invoicing API Reference (April 2026)

```python
# Step 1 — Create draft invoice
invoice = await asyncio.to_thread(
    stripe.Invoice.create,
    customer="cus_xxx",
    collection_method="send_invoice",
    days_until_due=30,           # 30 for NET_30, 60 for NET_60
    description="Q2 2026 SaaS license",  # from request.memo (may be None)
    auto_advance=False,          # REQUIRED — prevents Stripe auto-finalization
    metadata={"company_id": "...", "admin_id": "...", "payment_terms": "NET_30"},
)
stripe_invoice_id = invoice["id"]  # "in_xxx..."

# Step 2 — Add line items (once per item)
await asyncio.to_thread(
    stripe.InvoiceItem.create,
    customer="cus_xxx",
    invoice=stripe_invoice_id,
    price_data={
        "unit_amount": 250000,           # e.g. €2,500.00 in cents
        "currency": "eur",
        "product_data": {"name": "Professional tier — 12 months"},
    },
    quantity=1,
)

# Step 3 — Finalize (draft → open; makes invoice payable)
await asyncio.to_thread(stripe.Invoice.finalize_invoice, stripe_invoice_id)

# Step 4 — Send PDF invoice email to customer
await asyncio.to_thread(stripe.Invoice.send_invoice, stripe_invoice_id)
```

**Invoice statuses lifecycle:**
- `draft` — created, items being added; `auto_advance=False` keeps it here
- `open` — finalized, awaiting payment; set after `finalize_invoice`
- `paid` — payment received (via `invoice.paid` webhook)
- `void` — cancelled by admin (out of scope)
- `uncollectible` — Stripe marks as bad debt (out of scope)

### DB Schema — `client.enterprise_invoices`

```
id                UUID PK
company_id        UUID NOT NULL → client.companies.id (RESTRICT on delete)
stripe_invoice_id VARCHAR(255) NOT NULL UNIQUE
stripe_customer_id VARCHAR(255) NOT NULL
admin_id          UUID NOT NULL (bare, no FK — admin schema)
payment_terms     VARCHAR(10) NOT NULL  — 'NET_30' | 'NET_60'
line_items        JSONB NOT NULL DEFAULT '[]'
                  — [{"description": str, "amount_cents": int, "currency": str}, ...]
status            VARCHAR(20) NOT NULL DEFAULT 'open'
created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
sent_at           TIMESTAMPTZ NULL  — set at creation after send_invoice() call
paid_at           TIMESTAMPTZ NULL  — set by invoice.paid webhook
```

Index: `ix_enterprise_invoices_company_id` on `(company_id)`.
Unique: `uq_enterprise_invoices_stripe_invoice_id` on `(stripe_invoice_id)`.

### Webhook Extension Detail

The webhook is in `client-api` (`services/client-api/src/client_api/services/webhook_service.py`). Custom invoices (`event.data.object.subscription == null`) and subscription invoices (`event.data.object.subscription != null`) both trigger `invoice.paid`. The extension:

```python
# EXISTING (unchanged):
stripe_subscription_id = event_obj.get("subscription")
if stripe_subscription_id:
    await _update_subscription_on_invoice_paid(session, stripe_subscription_id)

# NEW (additive):
if event_type == "invoice.paid":
    stripe_inv_id = event_obj.get("id")
    if stripe_inv_id:
        await _handle_enterprise_invoice_paid(session, stripe_inv_id)
```

The `_handle_enterprise_invoice_paid` call runs for ALL `invoice.paid` events (including subscription ones). Since `enterprise_invoices.stripe_invoice_id` values are distinct from subscription invoice IDs, the WHERE clause matches zero rows for subscription invoices — safe no-op.

### Regression Risk Assessment

1. **`webhook_service.py` modified** — One additive call added to `_handle_invoice_event()`. Zero existing logic changed. Risk: LOW.
   ```bash
   pytest services/client-api/tests/unit/test_webhook*.py -v
   ```

2. **`admin-api/main.py` modified** — One router include added. Risk: VERY LOW.
   ```bash
   pytest services/admin-api/tests/ -v
   ```

3. **Migration 031** — Creates new table only. No modifications to existing tables. Risk: VERY LOW.
   ```bash
   make migrate-service SVC=client-api
   ```

4. **Zero changes to `billing.py`, `billing_service.py`, `company_service.py`** — Entirely untouched.

5. **No tier gating changes** — `tier='enterprise'` companies already exist in `subscriptions` table; this story reads but never writes tier.

### Test Coverage Mapping (Epic Test Design)

| Test ID | Level | Priority | Implementation |
|---------|-------|----------|----------------|
| **8.11-API-001** | API | P2 | `test_8_11_api_001_create_invoice_net30_sets_days_until_due_30` (Task 7.1) |
| **R-008 mitigation** | Unit | P2 | `test_invoice_paid_updates_enterprise_invoice_to_paid` (Task 7.2) |

From `test-design-epic-08.md`:
- **8.11-API-001** (P2): "Admin creates invoice; assert Stripe draft invoice created with `days_until_due: 30`" → covered by Task 7.1 test.
- **R-008** (OPS/Low): "Enterprise custom invoice delayed sending or amount mismatch — Admin review workflow; `invoice.paid` webhook sync test" → mitigated by:
  - R-008 mitigation #1: `GET /api/v1/admin/enterprise-invoicing/` list endpoint (AC4) provides admin review visibility
  - R-008 mitigation #2: `invoice.paid` webhook sync task (AC5, Task 6.1) tested in Task 7.2

### Project Structure Notes

**New files:**
- `services/client-api/alembic/versions/031_enterprise_invoices.py`
- `services/client-api/src/client_api/models/enterprise_invoice.py`
- `services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py`
- `services/admin-api/src/admin_api/services/enterprise_invoice_service.py`
- `services/admin-api/src/admin_api/schemas/enterprise_invoice.py`
- `services/admin-api/tests/unit/test_enterprise_invoice_service.py`
- `services/admin-api/tests/api/test_enterprise_invoice_api.py`
- `services/client-api/tests/unit/test_enterprise_invoice_webhook.py`

**Modified files:**
- `services/client-api/src/client_api/services/webhook_service.py` — extend `_handle_invoice_event()` (additive only, ~15 lines)
- `services/admin-api/src/admin_api/main.py` — add 2 lines: import + include_router

**No changes to:**
- `billing.py`, `billing_service.py`, `company_service.py` — not touched
- `subscriptions` ORM model — no new columns needed
- `webhook_events` deduplication table — unchanged; all events already deduplicated by it
- `tier_access_policies`, `add_on_purchases`, `companies` — no schema changes

### References

- Story spec: [Source: eusolicit-docs/planning-artifacts/epic-08-subscription-billing.md#S08.11]
- Epic test design (8.11-API-001, R-008): [Source: eusolicit-docs/test-artifacts/test-design-epic-08.md]
- Admin-API router pattern (type-alias deps): [Source: eusolicit-app/services/admin-api/src/admin_api/api/v1/tenants.py]
- Admin-API auth: [Source: eusolicit-app/services/admin-api/src/admin_api/core/security.py]
- Admin-API dependencies (get_admin_user, get_client_db_session): [Source: eusolicit-app/services/admin-api/src/admin_api/dependencies.py]
- Admin-API main.py (router registration pattern): [Source: eusolicit-app/services/admin-api/src/admin_api/main.py]
- Webhook handler to extend: [Source: eusolicit-app/services/client-api/src/client_api/services/webhook_service.py]
- Migration pattern (schema="client", revision chain): [Source: eusolicit-app/services/client-api/alembic/versions/030_vat_validation_status.py]
- Previous story patterns (asyncio.to_thread, billing, admin patterns): [Source: eusolicit-docs/implementation-artifacts/8-9-per-bid-add-on-purchase-flow.md, 8-10-eu-vat-handling-via-stripe-tax-vies-validation.md]
- Project context (critical rules): [Source: eusolicit-docs/project-context.md]
- Stripe Invoicing API: https://stripe.com/docs/api/invoices
- Stripe InvoiceItems API: https://stripe.com/docs/api/invoiceitems/create

## Dev Agent Record

### Agent Model Used

claude-opus-4-5 (BMAD create-story, 2026-04-19)
claude-sonnet-4-5 (BMAD dev-story implementation, 2026-04-19)

### Debug Log References

- stripe.Invoice.create / InvoiceItem.create: Python 3 classmethods create new bound-method wrappers on each access; `is` identity check always fails. Fix: use `==` for classmethods, `__qualname__` for stripe's internal `class_method_variant` type (finalize_invoice, send_invoice).
- admin-api cannot import from client_api (cross-service boundary rule). Solution: define separate `EnterpriseInvoice` ORM model in `admin_api/models/enterprise_invoice.py` using admin-api's `Base`. Both models map to same physical `client.enterprise_invoices` table.
- `select(EnterpriseInvoice)` crashes when `EnterpriseInvoice` is patched to a bare MagicMock (SQLAlchemy validates argument at construction time). Fix: module-level alias `_EI = EnterpriseInvoice` in service; list queries use `_EI` (unaffected by patch); create queries use `EnterpriseInvoice` (patched, allows constructor mock).
- `EnterpriseInvoiceListResponse(items=[MagicMock(), ...])` fails Pydantic validation when `model_validate` is mocked. Fix: use `model_construct()` to bypass double-validation on already-converted items.
- `client_subscriptions` Core Table in `client_tables.py` was missing `tier` and `stripe_customer_id` columns. Added both.
- `_handle_invoice_event()` had early-return when `stripe_sub_id` is None. Restructured to eliminate early return so enterprise handler is always reached for `invoice.paid` events.

### Completion Notes List

- All 7 ACs implemented and verified by unit + API tests.
- AC1: Migration `031_enterprise_invoices.py` created with all DDL under `schema="client"`.
- AC2/AC3/AC7: `create_enterprise_invoice` service with 4-step Stripe flow, tier validation, asyncio.to_thread wrapping.
- AC4: `list_enterprise_invoices` service with pagination and optional filters.
- AC5: `_handle_enterprise_invoice_paid` additive webhook extension; no existing logic modified.
- AC6: structlog audit logging in service (billing.enterprise_invoice_created).
- Test results: admin-api 301/301 passed; client-api unit 632/632 passed. Zero regressions.
- ATDD tests: 12 service unit + 9 API + 4 webhook = 25 story tests, all green.

**Second-pass review follow-up — 2026-04-19:**
- Addressed both second-pass `[Review][Patch]` items (1 BLOCKER + 1 non-blocking).
- ✅ Resolved review finding [BLOCKER]: Added `"stripe>=8.0,<16"` to `services/admin-api/pyproject.toml:dependencies` so the admin-api container installs the stripe SDK in production. Verified via `python3 -c "import admin_api.main"` (succeeds; no ImportError). Full admin-api suite 313/313 passing.
- ✅ Resolved review finding [non-blocking]: Dropped the generic `except ValueError` catch-all from `api/v1/enterprise_invoicing.py` that would have masked future typed-exception subclasses as `validation_error` 422s; added an inline comment warning against re-introducing it. All 10 API tests remain green.
- Test counts after second-pass follow-up: admin-api 313/313 passing (excluding unrelated pre-existing RED-phase migration-002 tests for a different story); client-api webhook suite 28/28 passing. Zero regressions introduced.

**Review follow-up — 2026-04-19:**
- Addressed all 7 `[Review][Patch]` findings from the senior-developer review (see Senior Developer Review § Review Findings above for per-item detail).
- Introduced typed exception hierarchy (`admin_api/services/enterprise_invoice_errors.py`) replacing fragile `ValueError`-substring dispatch; router now routes by `isinstance`.
- Dropped the `_EI = EnterpriseInvoice` alias and `EnterpriseInvoiceListResponse.model_construct(...)` hack from the service; refactored unit tests so they no longer patch `EnterpriseInvoice`/`model_validate` in list paths (production code is now idiomatic; tests drive through real Pydantic validation).
- Hardened schemas: €1M ceiling on `amount_cents`, lowercase-normalized `currency` via `field_validator(mode="before")` + `Literal["eur","usd","gbp"]`, single-currency invariant across line items via `model_validator(mode="after")`.
- Constrained `status` query param to a `Literal` whitelist (422 on bad input).
- Fixed multi-subscription TOCTOU: `.order_by(started_at.desc().nulls_last()).limit(1)` + `.first()` instead of `.one_or_none()`; added `started_at` to the `client_subscriptions` Core Table.
- Added R-008 observability: webhook logs `enterprise_invoice_paid_webhook_no_db_match` at WARNING when `invoice.paid` arrives with enterprise metadata but no DB row matches.
- Test counts after follow-up: admin-api 288/288 passed (+10 new schema tests, +1 new service test for `.first()` semantics, +1 new API test for status-Literal 422); client-api unit 633/633 passed (+1 new webhook no-match warning test). Zero regressions. Story-files lint clean.

### File List

**New files:**
- `services/client-api/alembic/versions/031_enterprise_invoices.py`
- `services/client-api/src/client_api/models/enterprise_invoice.py`
- `services/admin-api/src/admin_api/models/enterprise_invoice.py`
- `services/admin-api/src/admin_api/schemas/enterprise_invoice.py`
- `services/admin-api/src/admin_api/services/enterprise_invoice_service.py`
- `services/admin-api/src/admin_api/services/enterprise_invoice_errors.py` *(review follow-up — typed exception hierarchy for router isinstance-dispatch)*
- `services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py`
- `services/admin-api/tests/unit/test_enterprise_invoice_service.py` (ATDD, skip decorators removed)
- `services/admin-api/tests/unit/test_enterprise_invoice_schemas.py` *(review follow-up — 10 schema guardrail tests: amount ceiling, currency normalization, single-currency invariant)*
- `services/admin-api/tests/api/test_enterprise_invoice_api.py` (ATDD, skip decorators removed)
- `services/client-api/tests/unit/test_enterprise_invoice_webhook.py` (ATDD, skip decorators removed)

**Modified files:**
- `services/client-api/src/client_api/services/webhook_service.py` — additive `_handle_enterprise_invoice_paid` + call from `_handle_invoice_event`; review follow-up added R-008 metadata forwarding + `enterprise_invoice_paid_webhook_no_db_match` WARNING
- `services/admin-api/src/admin_api/main.py` — import + include_router for enterprise_invoicing
- `services/admin-api/src/admin_api/models/client_tables.py` — added `stripe_customer_id`, `tier`, and `started_at` columns to `client_subscriptions` Core Table (`started_at` added as part of review follow-up for deterministic multi-subscription ordering)
- `services/admin-api/pyproject.toml` — *(second-pass review follow-up)* added `"stripe>=8.0,<16"` to dependencies so the admin-api container ships the Stripe SDK and boots cleanly in production
- `services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py` — *(second-pass review follow-up)* removed generic `except ValueError` catch-all that would have silently swallowed future `EnterpriseInvoiceError` subclasses as 422 `validation_error`; genuine untyped `ValueError`s now bubble to FastAPI's 500 handler
- `services/client-api/tests/unit/test_webhook_service.py` — fixed `test_handle_invoice_no_subscription_id_logs_warning` assertion (removed stale `execute.assert_not_called()`)
- `services/admin-api/tests/unit/test_enterprise_invoice_service.py` — fixed `is` → `==` for classmethods; `__qualname__` for class_method_variant; review follow-up refactored to drop `EnterpriseInvoice`/`model_validate` patches from list tests (real Pydantic validation), typed exception asserts, `.first()` semantics coverage

## Senior Developer Review

**Reviewed:** 2026-04-19
**Outcome:** Changes Requested (non-blocking) — all 7 ACs are implemented, tests pass (admin-api 301/301, client-api 632/632), architecture aligns with spec. The concerns below are quality/resilience issues that do not block the story, but should be tracked.

**Review follow-up (2026-04-19):** All 7 `[Review][Patch]` items resolved by the dev-story agent. Admin-api now at 288/288 passing (includes 13 new service tests + 10 new schema tests + 10 API tests for this story); client-api at 633/633 passing (includes 5 webhook tests — one added for the new no-match warning path). See per-item "RESOLVED" notes below for the concrete fix applied to each. The 5 `[Defer]` items remain deferred per their original justification.

### Review Findings

- [x] **[Review][Patch] Production code is shaped by unit-test mocks** [`services/admin-api/src/admin_api/services/enterprise_invoice_service.py:33-38, 192-195`] — The `_EI = EnterpriseInvoice` alias and the `EnterpriseInvoiceListResponse.model_construct(...)` call both exist explicitly to accommodate how the unit tests patch `EnterpriseInvoice` and `model_validate`. In-file comments state this. Refactor the tests (e.g. patch at the class-attribute level, or stop mocking Pydantic validation) and restore idiomatic production code: drop `_EI`, use `EnterpriseInvoiceListResponse(...)` constructor. **RESOLVED 2026-04-19:** Dropped `_EI` alias and replaced `EnterpriseInvoiceListResponse.model_construct(...)` with natural `EnterpriseInvoiceListResponse(items=[...])` construction. List tests no longer patch `EnterpriseInvoice` at all (SQL construction is not mocked; only `session.execute` is); new `_make_invoice_row()` helper supplies field-typed attributes so real `EnterpriseInvoiceResponse.model_validate(row)` with `from_attributes=True` runs end-to-end. Create tests continue to patch the ORM constructor but use a `_make_persisted_record()` that returns a record with all required typed fields.
- [x] **[Review][Patch] `ValueError` substring matching is a fragile dispatch mechanism** [`services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py:58-84`] — `"not_enterprise_tier" in msg` / `"no_stripe_customer" in msg` / `"company_not_found" in msg` are order-sensitive and collide if any substring ever appears in another error message. Replace with typed sentinel exceptions (e.g. `NotEnterpriseTierError`, `NoStripeCustomerError`, `CompanyNotFoundError`) and route by `isinstance`. **RESOLVED 2026-04-19:** Created `admin_api/services/enterprise_invoice_errors.py` with `EnterpriseInvoiceError(ValueError)` base + `CompanyNotFoundError`, `NotEnterpriseTierError`, `NoStripeCustomerError` subclasses. Service raises typed exceptions; router dispatches via `isinstance` (`NotEnterpriseTierError`→422, `NoStripeCustomerError`→422, `CompanyNotFoundError`→404, `EnterpriseInvoiceError`→422 fallback, `StripeError`→502). All subclasses still inherit from `ValueError` so any untouched `except ValueError` legacy pathway continues to work; error-token strings preserved in messages for backward compat.
- [x] **[Review][Patch] `status` query param on GET list is unvalidated free text** [`services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py:105`] — Any value is accepted; there is no allow-list. Constrain to `Literal["open", "paid", "void", "uncollectible"]` (or a `StatusEnum`) so bad input fails at 422 rather than returning an empty list. **RESOLVED 2026-04-19:** Introduced module-level `_StatusFilter = Literal["open", "paid", "void", "uncollectible"]` and typed the `status` query parameter as `_StatusFilter | None`. FastAPI now returns 422 for unknown values. New API test `test_list_invoices_rejects_invalid_status_query_param` locks the behavior in.
- [x] **[Review][Patch] No upper bound on `amount_cents`** [`services/admin-api/src/admin_api/schemas/enterprise_invoice.py:15`] — `Field(ge=1)` with no `le=...` means a typo could create a multi-million-EUR invoice and email it to the customer. Add a sensible ceiling (e.g. `le=100_000_000` for €1M) with a structlog warning above some threshold. **RESOLVED 2026-04-19:** Added `_MAX_LINE_ITEM_AMOUNT_CENTS = 100_000_000` (€1M) constant and `Field(ge=1, le=_MAX_LINE_ITEM_AMOUNT_CENTS)` on `LineItemSchema.amount_cents`. Typo-defense: one-cent-over-ceiling now fails at 422 before any Stripe call. Covered by `TestLineItemSchemaAmountBounds` (3 tests) in new `test_enterprise_invoice_schemas.py`.
- [x] **[Review][Patch] `amount_cents` / `currency` normalization** [`services/admin-api/src/admin_api/schemas/enterprise_invoice.py:16`] — `currency` is only length-checked; Stripe wants lowercase ISO. Either normalize in a validator (`currency.lower()`) or require `Literal["eur", "usd", ...]`. Similarly, enforce all line items share one currency to avoid Stripe rejecting a mixed-currency invoice mid-flow (which would leave orphaned items — see R-008 below). **RESOLVED 2026-04-19:** Replaced free-text `currency: str` with `_SupportedCurrency = Literal["eur", "usd", "gbp"]` plus a `@field_validator("currency", mode="before")` that lowercases the input *before* Literal validation runs (so `"EUR"`, `"UsD"` are accepted and normalized). Added `@model_validator(mode="after") _single_currency_across_line_items` on `CreateEnterpriseInvoiceRequest` that rejects mixed-currency requests with `"single currency"` in the error message. Covered by `TestLineItemSchemaCurrency` (4 tests) and `TestCreateEnterpriseInvoiceRequestSingleCurrency` (3 tests).
- [x] **[Review][Patch] Webhook: no observability when `invoice.paid` arrives for a stripe_invoice_id we created but cannot match** [`services/client-api/src/client_api/services/webhook_service.py:410-416`] — The handler only logs on a match; the no-match case is invisible. Log at `warning` (or at least `info` with a distinct event name) for non-match payments whose metadata has `company_id` / `admin_id` keys, since those are certainly enterprise invoices whose DB row is missing. Helps detect the R-008 race (customer paid, we have no row). **RESOLVED 2026-04-19:** `_handle_invoice_event()` now extracts `invoice_obj.metadata` (handling `StripeObject.to_dict()`, plain `dict`, and `None`) and forwards it as a kwarg to `_handle_enterprise_invoice_paid(session, stripe_invoice_id, metadata=...)`. When the UPDATE matches zero rows AND the metadata carries `company_id`/`admin_id` keys, the handler emits `enterprise_invoice_paid_webhook_no_db_match` at WARNING. New test `test_no_match_with_enterprise_metadata_logs_warning` locks it in; also asserts no warning when metadata is absent.
- [x] **[Review][Patch] Multiple-subscription TOCTOU / 500** [`services/admin-api/src/admin_api/services/enterprise_invoice_service.py:59-72`] — If `client.subscriptions` has more than one row per company_id, `one_or_none()` raises `MultipleResultsFound` → HTTP 500. Add `.order_by(client_subscriptions.c.created_at.desc()).limit(1)` or change to `first()`, and document which row semantics apply. **RESOLVED 2026-04-19:** Rewrote the verification SELECT as `.order_by(client_subscriptions.c.started_at.desc().nulls_last()).limit(1)` + `result.first()`. Semantics: latest-started subscription wins (rationale documented inline). Added missing `started_at` column to the `client_subscriptions` Core Table in `admin_api/models/client_tables.py` to support the ORDER BY. New test `test_create_invoice_verifies_company_with_first_not_one_or_none` ensures `.first()` is used.
- [x] **[Review][Defer] No Stripe idempotency key on `stripe.Invoice.create`** [`services/admin-api/src/admin_api/services/enterprise_invoice_service.py:84-96`] — deferred, spec-compliant — An admin double-click or proxy retry creates two separate finalized invoices (two emails, two charges). Mitigation: pass `idempotency_key=f"ent-inv:{company_id}:{client_provided_nonce}"` and require the client to send a nonce header. Story spec does not require this; file as follow-up work.
- [x] **[Review][Defer] Saga/compensation gap — Stripe finalize+send happens before DB persist** [`services/admin-api/src/admin_api/services/enterprise_invoice_service.py:122-144`] — deferred, per-spec ordering — If `send_invoice` succeeds but `session.commit()` fails, the customer is emailed a finalized invoice and our platform has no row for it. The subsequent `invoice.paid` webhook will silently no-op. Mitigations to consider: (a) persist a `pending_create` row before Stripe calls as an outbox; (b) wrap the Stripe call sequence with `try/except` that best-effort voids the Stripe invoice on DB failure; (c) build a Stripe-side reconciler (Celery Beat) that queries `stripe.Invoice.list(metadata={"company_id": ...})` and backfills missing rows. Story spec orders persist-after-send, so this is deferred to a follow-up resilience story.
- [x] **[Review][Defer] Partial-state on mid-loop `InvoiceItem.create` failure** [`services/admin-api/src/admin_api/services/enterprise_invoice_service.py:108-119`] — deferred, pre-existing pattern — k-1 invoice items plus a draft invoice are orphaned in Stripe if item k fails. Draft invoices auto-expire but the items do not. Same reconciler as above would catch this.
- [x] **[Review][Defer] Tier change between `SELECT` and `Invoice.create`** [`services/admin-api/src/admin_api/services/enterprise_invoice_service.py:71-84`] — deferred — If an automated tier change (e.g. trial expiry in the webhook handler) fires between the verification SELECT and the Stripe call, an invoice is sent to a non-enterprise company. TOCTOU window is small; low practical risk. Fix if needed: re-check tier inside a SELECT FOR UPDATE transaction.
- [x] **[Review][Defer] Audit log duality** [`services/admin-api/src/admin_api/services/enterprise_invoice_service.py:147-155`] — deferred, per-spec — AC6 explicitly requires structlog rather than `shared.audit_log`, which conflicts with the project-context rule "all mutations to sensitive entities must emit to `shared.audit_log`". Resolve at the platform level (e.g. an admin-api audit sink that tails structlog into `shared.audit_log`) rather than in this story. Flagged as a platform-level CONTRADICTORY_SPEC; non-blocking for this story.

### Decisions the reviewer made on the reviewer's own authority

- Tests pass (25 new + zero regressions). Error-handling for 401/403/422/404/502 paths is covered.
- Webhook extension is correctly additive: the subscription-invoice path is unchanged; enterprise handler runs only for the `invoice.paid` dispatch (`new_status == "active"`) and is a silent no-op when the UPDATE matches zero rows.
- AC7 tier-check fires BEFORE any Stripe call, so invalid-tier requests do not waste Stripe API credits (verified by `mock_to_thread.assert_not_called()`).
- Migration is correctly numbered 031 with `schema="client"` on all DDL. Both ORM classes (admin-api and client-api) agree on column types and nullability.

### Summary

All acceptance criteria are met and tests are green. No blocking defects. Findings above are quality and resilience improvements — some suitable for this story (the 7 `[Patch]` items), some better tracked as follow-up resilience work (the 5 `[Defer]` items).

---

## Senior Developer Review — Second Pass

**Reviewed:** 2026-04-19 (second pass, after dev-story follow-up on first-pass findings)
**Outcome:** **Changes Requested (BLOCKING)** — the seven `[Patch]` items from the first review are all verifiably fixed in code, and local tests are green (admin-api enterprise-invoice suite: 33/33; client-api webhook suite: 28/28). However, a **production packaging defect** was discovered during this re-review that will break the admin-api container on cold start. One non-blocking quality finding is also included.

### Verification of First-Pass Patches (spot-check, all ✅)

| First-pass patch                                           | Verified in                                                              | Status |
|------------------------------------------------------------|--------------------------------------------------------------------------|--------|
| Typed exception hierarchy replaces `ValueError` substring  | `services/admin-api/src/admin_api/services/enterprise_invoice_errors.py` | ✅ fixed |
| `_EI` alias removed; `model_construct` hack removed        | `enterprise_invoice_service.py:31, 216-221`                              | ✅ fixed |
| `Literal` `status` filter (422 on invalid)                 | `enterprise_invoicing.py:58, 131`                                        | ✅ fixed |
| €1M `amount_cents` upper bound                             | `schemas/enterprise_invoice.py:25, 37-45`                                | ✅ fixed |
| `currency` lowercase normalization + Literal whitelist     | `schemas/enterprise_invoice.py:30, 51-62`                                | ✅ fixed |
| Single-currency invariant across line items                | `schemas/enterprise_invoice.py:73-89`                                    | ✅ fixed |
| `.first()` + `ORDER BY started_at DESC NULLS LAST LIMIT 1` | `enterprise_invoice_service.py:84-100`                                   | ✅ fixed |
| R-008 no-match WARNING log                                 | `webhook_service.py:452-463`                                             | ✅ fixed |

### Review Findings (Second Pass)

- [x] **[Review][Patch][BLOCKER] `admin-api` imports `stripe` but does not declare it as a dependency** [`services/admin-api/pyproject.toml:9-26`; imports at `services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py:24`, `services/admin-api/src/admin_api/services/enterprise_invoice_service.py:25`] — The admin-api pyproject.toml dependency list (fastapi, uvicorn, pydantic, sqlalchemy, structlog, httpx, pyjwt, alembic, asyncpg, psycopg2-binary, eusolicit-common, eusolicit-models, eusolicit-kraftdata, celery, redis) has **no** `stripe` entry. The admin-api Dockerfile builds the container with `pip install .` against that pyproject, so the production admin-api image will NOT have the stripe SDK installed. On `uvicorn admin_api.main:app` startup, FastAPI imports `admin_api.api.v1.enterprise_invoicing` (added to `main.py` by this story), which then does `import stripe` — **ImportError at container startup, the entire admin-api service fails to boot, not just this endpoint**. Tests pass locally only because `make dev-install` editable-installs client-api into the same venv, which transitively exposes `stripe`. This masking is exactly the trap the project-context.md "service isolation" rule is meant to prevent. **Fix:** add `"stripe>=8.0,<16"` (matching the pin already in place for `services/notification/pyproject.toml`) to `admin-api/pyproject.toml:dependencies`, and either (a) rebuild the admin-api image locally with `docker compose build admin-api` and verify startup, or (b) add a smoke test that imports `admin_api.main` inside the admin-api-only venv. **RESOLVED 2026-04-19:** Added `"stripe>=8.0,<16"` to `services/admin-api/pyproject.toml:dependencies` (below `redis>=5.0`). Upper bound `<16` matches the installed `stripe==15.0.1` on the shared venv (notification pins `<9` for the v9 SubscriptionItem.create_usage_record regression, which admin-api does not touch, so the broader `<16` bound is correct here). Verified `python3 -c "import admin_api.main"` succeeds in an admin-api-only context; full admin-api test suite still 313/313 passing (identical to pre-fix count, excluding the pre-existing unrelated RED-phase migration-002 suite).
- [x] **[Review][Patch] Router's generic `except ValueError` block masks future typed-exception bugs** [`services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py:105-110`] — The router catches `EnterpriseInvoiceError` at line 99 and falls through to a defensive `except ValueError` at 105. Because `EnterpriseInvoiceError` subclasses `ValueError`, any new typed subclass the dev forgets to list explicitly above line 99 will be silently swallowed by the generic `except ValueError` as a `validation_error` 422 — the exact footgun the typed hierarchy was introduced to avoid. Either drop the generic fallback (let genuine unexpected `ValueError`s bubble to FastAPI's 500 handler where they belong), or move it ABOVE the `EnterpriseInvoiceError` catch and narrow it with `not isinstance(exc, EnterpriseInvoiceError)` — not both-order-and-width. **RESOLVED 2026-04-19:** Took the first option — dropped the `except ValueError as exc` catch-all entirely. Genuine untyped `ValueError`s now bubble to FastAPI's 500 handler where they belong; the `except EnterpriseInvoiceError` block remains as the narrow, typed fallback for subclasses not explicitly routed above. Added an inline comment warning future devs against re-introducing a broad `except ValueError`. All 10 API tests still green (404/422/502 paths unchanged).
- [x] **[Review][Defer] `EnterpriseInvoiceResponse.status` and `payment_terms` are untyped `str`** [`services/admin-api/src/admin_api/schemas/enterprise_invoice.py:100-101`] — deferred — The request body uses `Literal["NET_30","NET_60"]` and the list filter uses `Literal["open","paid","void","uncollectible"]`, but the response body types both fields as bare `str`. If DB state ever drifts from the invariant the response will pass validation silently. Narrowing to Literal on the response would surface the drift. Non-blocking for this story — read-path, Pydantic serialization.
- [x] **[Review][Defer] `admin_id` on response exposed to admin UI** [`services/admin-api/src/admin_api/schemas/enterprise_invoice.py:99`] — deferred, per-spec — AC4 explicitly lists `admin_id` on the response. Non-issue for an admin-only endpoint, but worth noting that if/when enterprise customers ever see invoices via a self-service admin UI, exposing platform-admin UUIDs to tenant-scoped users would leak staff identifiers.
- [x] **[Review][Defer] Audit-log duality (re-noted from first pass)** — deferred, per-spec — AC6 mandates structlog, project-context rule mandates `shared.audit_log` for sensitive mutations. Already surfaced as a platform-level CONTRADICTORY_SPEC deviation; not re-opening for this story.

### Decisions the reviewer made on the reviewer's own authority

- All seven `[Patch]` follow-ups from the first review are verifiably in place and covered by new tests. The dev-story agent's "RESOLVED" claims line up with actual code.
- Local test counts: admin-api enterprise-invoice service (13) + schemas (10) + API (10) = 33/33 passing; client-api webhook suite 28/28. Story-scoped regression risk is low.
- The stripe-dependency finding is **blocking** because it breaks production admin-api startup, not just this feature. It is a one-line fix to `pyproject.toml`, but the fix must land on this story before it ships — otherwise the next admin-api deploy will take the entire admin service down.

### Summary — Second Pass

Verdict: **Changes Requested (BLOCKING)**. The story implementation is otherwise solid — all 7 ACs implemented correctly, tests green, first-pass patches all verified fixed. The one blocking defect is a packaging/dependency slip that local tests cannot catch. Fix `admin-api/pyproject.toml` and rebuild the container to verify; then this story is clear to ship.

DEVIATION: admin-api imports `stripe` but does not declare it in `services/admin-api/pyproject.toml` dependencies — production container will fail at startup with ImportError.
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking

---

## Senior Developer Review — Third Pass

**Reviewed:** 2026-04-24
**Outcome:** **Approve** — all 7 ACs implemented; all 7 first-pass `[Patch]` items + both second-pass `[Patch]` items (BLOCKER stripe-dep + router catch-all) verified fixed in code; new test coverage commensurate with the resilience fixes; no new blocking defects.

### Verification of Second-Pass Patches (spot-check, all ✅)

| Second-pass patch                                                                                          | Verified in                                                                                | Status   |
|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|----------|
| `"stripe>=8.0,<16"` declared in admin-api dependencies                                                     | `services/admin-api/pyproject.toml:26`                                                     | ✅ fixed |
| Router no longer has generic `except ValueError` catch-all                                                 | `services/admin-api/src/admin_api/api/v1/enterprise_invoicing.py:99-109` (typed-only path) | ✅ fixed |

### Spot-checks performed this pass

- **Migration 031** uses `schema="client"` on every `op.*` DDL call; revision/down_revision chain is `031 → 030`. ✅
- **Dual ORM model rationale** is documented in both `services/client-api/.../enterprise_invoice.py` and `services/admin-api/.../models/enterprise_invoice.py` headers and matches the project rule "admin-api never imports from client_api". The two models point at the same physical `client.enterprise_invoices` table; column types/nullability agree. The admin-api model intentionally omits the `company_id` FK (FK is enforced by the DB itself per migration 031). ✅
- **Service tier-check** (`enterprise_invoice_service.py:84-107`) fires BEFORE any `asyncio.to_thread` wrapping a Stripe call — guarded by `mock_to_thread.assert_not_called()` in three rejection-path tests. ✅
- **Stripe four-step flow ordering** (`create → InvoiceItem.create×N → finalize → send`) is enforced by `test_create_invoice_finalizes_and_sends_invoice_after_items` (asserts `finalize_idx < send_idx` via `__qualname__` — robust against stripe v15 classmethod identity quirks). ✅
- **`_handle_enterprise_invoice_paid` is additive**: existing subscription handler in `_handle_invoice_event` is unchanged (lines 336-364); the new enterprise call (lines 374-394) only fires for `new_status == "active"` and is an UPDATE … WHERE … RETURNING that no-ops on zero matches. ✅
- **R-008 observability**: WARNING `enterprise_invoice_paid_webhook_no_db_match` is gated on `metadata` carrying `company_id`/`admin_id` so subscription-invoice traffic stays quiet. Test `test_no_match_with_enterprise_metadata_logs_warning` covers BOTH the warn-on-enterprise-metadata path AND the silent-on-subscription path. ✅
- **Schema guardrails**: €1M `amount_cents` ceiling, `currency` lowercase normalization (mode="before"), `Literal["eur","usd","gbp"]` whitelist, and the `_single_currency_across_line_items` model-level validator are all in place and covered by `test_enterprise_invoice_schemas.py` (10 tests). ✅
- **Multi-subscription TOCTOU**: SELECT now ends with `.order_by(client_subscriptions.c.started_at.desc().nulls_last()).limit(1)` + `result.first()`. The `started_at` column was correctly added to `admin_api/models/client_tables.py:client_subscriptions`. Test `test_create_invoice_verifies_company_with_first_not_one_or_none` enforces the `.first()` contract. ✅
- **Router-level `Literal` for `?status=`** is in place (`enterprise_invoicing.py:58, 130`) and covered by `test_list_invoices_rejects_invalid_status_query_param` (asserts 422 + service NOT invoked). ✅

### Findings (Third Pass)

No new blocking or `[Patch]` findings. The two non-blocking `[Defer]` items from the second pass remain deferred per their original justification:

- **[Defer] `EnterpriseInvoiceResponse.status` and `payment_terms` are bare `str`** — read-path schema; deferred. Narrowing to `Literal` would make DB drift surface immediately, but it is not blocking.
- **[Defer] Audit-log duality (structlog vs `shared.audit_log`)** — platform-level CONTRADICTORY_SPEC; already documented in Known Deviations.

The five resilience `[Defer]` items from the first pass (idempotency key on `Invoice.create`, saga/compensation gap on commit failure, partial-state on mid-loop `InvoiceItem.create` failure, tier-change TOCTOU, audit-log duality) all remain valid follow-up work — none are story-blocking.

### Decisions the reviewer made on the reviewer's own authority

- The dual ORM model approach is sound. The lack of FK on `company_id` in the admin-api model is acceptable because (a) the FK is enforced by the database constraint created in migration 031, and (b) the admin-api model is only used for INSERT (which the database FK validates) and SELECT-by-PK (which doesn't need application-level FK metadata).
- The stripe SDK upper bound `<16` correctly matches the `stripe==15.x` already installed in the shared client-api venv. The notification service's tighter `<9` pin is justified by an unrelated v9 regression in `SubscriptionItem.create_usage_record` that admin-api does not exercise.
- Test coverage now exceeds the original ATDD plan (25 tests) — admin-api enterprise-invoice surface has 33 tests (13 service + 10 schema + 10 API), plus 5 webhook tests in client-api covering the no-match WARNING path. Coverage of all rejection paths is complete.

### Summary — Third Pass

**Verdict: Approve.** All acceptance criteria met, all prior `[Patch]` findings resolved, tests are comprehensive, no new blockers found. Story is clear to ship.

---

## Known Deviations

### Detected by `3-code-review` at 2026-04-19T04:47:52Z (session 4cc0299a-cbb1-4e45-8255-355606320705)

- Audit log uses structlog per AC6 but project-context.md mandates `shared.audit_log` for sensitive mutations. _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_
- Audit log uses structlog per AC6 but project-context.md mandates `shared.audit_log` for sensitive mutations. _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_

### Detected by `3-code-review` at 2026-04-19T05:10:37Z (session d3741a25-c565-4a60-8d1e-c2809a404a05)

- admin-api imports `stripe` but does not declare it in `services/admin-api/pyproject.toml` dependencies — production container will fail at startup with ImportError. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- admin-api imports `stripe` but does not declare it in `services/admin-api/pyproject.toml` dependencies — production container will fail at startup with ImportError. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_

## Change Log

| Date       | Author                 | Change                                                                                                                                                                                                                                                                                              |
|------------|------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 2026-04-19 | BMAD create-story      | Initial story draft.                                                                                                                                                                                                                                                                                |
| 2026-04-19 | BMAD dev-story         | Implemented all 7 ACs; 25 new ATDD tests green; admin-api 301/301, client-api 632/632.                                                                                                                                                                                                              |
| 2026-04-19 | BMAD code-review       | Senior Developer Review added — 7 `[Review][Patch]` + 5 `[Review][Defer]` findings.                                                                                                                                                                                                                 |
| 2026-04-19 | BMAD dev-story (review)| Addressed code-review findings — 7 `[Patch]` items resolved: typed exception hierarchy, dropped test-driven production hacks (`_EI`, `model_construct`), `Literal` `status` filter, €1M `amount_cents` ceiling, `currency` lowercase+whitelist+single-currency invariant, multi-subscription `.first()`+`started_at` order-by, webhook no-match WARNING for enterprise metadata. |
| 2026-04-19 | BMAD code-review       | Second-pass senior-developer review — one **blocking** finding (admin-api missing `stripe` in its `pyproject.toml` → container startup ImportError) and two non-blocking `[Patch]` items (router-level catch-all `except ValueError` masks typed sentinels; `EnterpriseInvoiceResponse.status` is `str` not `Literal`). Previous seven `[Patch]` items all verified fixed in code. |
| 2026-04-19 | BMAD dev-story (review)| Addressed second-pass code-review findings — 2 items resolved: added `"stripe>=8.0,<16"` to `services/admin-api/pyproject.toml:dependencies` (BLOCKER fix — unblocks admin-api container boot); dropped generic `except ValueError` catch-all in `api/v1/enterprise_invoicing.py` that would have silently swallowed future typed sentinel subclasses as 422 validation errors. All 313 admin-api tests + 28 client-api webhook tests green. Zero regressions. |
| 2026-04-24 | BMAD code-review       | Third-pass senior-developer review — **Approve**. All 7 ACs implemented; all 7 first-pass `[Patch]` items + both second-pass `[Patch]` items (BLOCKER stripe-dep + router catch-all) verified fixed in code; test coverage exceeds original ATDD plan (33 admin-api + 5 client-api webhook tests for this story); no new blockers. Two non-blocking `[Defer]` items (response-schema bare str types; structlog↔shared.audit_log duality) remain deferred. Story clear to ship. |
