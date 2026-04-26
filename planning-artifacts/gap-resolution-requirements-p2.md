# EU Solicit — Gap Resolution Requirements: P2 Gaps

**Date**: 2026-04-22 | **Author**: Mary (Business Analyst) | **Status**: Draft for review
**Purpose**: Exhaustive requirements for P2 gaps (GAP-05, 08, 09, 11, 12, 13, 14, 15) in the form needed for Requirements Brief v5, PRD v2, and scope-extension story drafting.
**Decisions applied**: (1) all watchlist items become proposed stories, (2) the three P1 design spikes NS-01.6 / NS-02.5 / NS-10.6 go to PRD §7 Out-of-Scope, (3) NS-07.4 + NS-07.5 proceed.

---

## GAP-05 — ESPD Auto-Fill

### Rationale (strategic)
ESPD is a mandatory document for EU public procurement tenders above threshold values. Pre-filling it from stored company data is a high-value automation for all paid tiers. Coverage is ✅ delivered via stories 2-12, 11-1, 11-2, 11-3, 11-13.

### Data model

**DM-05.1** `client.espd_profiles`:
- `id` UUID PK
- `company_id` UUID FK → `client.companies.id` (CASCADE)
- `profile_name` VARCHAR(255) NOT NULL
- `espd_data` JSONB NOT NULL default `{}` — structured by ESPD Part: `{part_ii, part_iii, part_iv, part_v}`
- `created_at`, `updated_at` TIMESTAMPTZ
- Index on `company_id`

ESPD Part semantics (EU-aligned):
- **Part II**: Operator identification (name, VAT, address, size, representative)
- **Part III**: Exclusion grounds (criminal, payment, insolvency, professional misconduct)
- **Part IV**: Selection criteria (suitability, economic capacity, technical capacity)
- **Part V**: Reduction of qualified candidates

### API contracts

**API-05.1 ESPD Profile CRUD** (`/api/v1/espd-profiles`)
| Method | Path | Role | Success | Errors |
|--------|------|------|---------|--------|
| POST | `/` | admin, bid_manager | 201 profile | 422 invalid body, 403 role |
| GET | `/` | any | 200 `{profiles[], total}` | — |
| GET | `/{id}` | any | 200 profile | 404 cross-tenant/missing |
| PATCH | `/{id}` | admin, bid_manager | 200 profile | 404, 403 |
| DELETE | `/{id}` | admin, bid_manager | 204 | 404, 403 |

**API-05.2 Auto-Fill** (`/api/v1/espd-profiles/{profile_id}/auto-fill`)
- POST, body `{opportunity_id}`. Returns `{espd_data, snapshot_profile_id, changed_fields[], opportunity_id, source_profile_id}`.
- Creates a new profile snapshot `"Auto-fill snapshot: {original_name}"`; does not mutate the source.
- Role: admin / bid_manager.

**API-05.3 Export** (`/api/v1/espd-profiles/{profile_id}/export`)
- POST `?format=xml|pdf`. Returns `StreamingResponse` with `Content-Type: application/xml` or `application/pdf`, filename `espd-{profile_id}.{ext}`.
- Role: admin / bid_manager.

### Functional requirements

**FR-05.1 Tenant isolation** — `company_id` is derived exclusively from JWT; any `company_id` in request body is ignored. Cross-tenant GET/PATCH/DELETE returns 404 to prevent UUID enumeration.

**FR-05.2 PATCH merge semantics** — `espd_data` PATCH is top-level Part merge, not recursive: providing `{"part_iii": {...}}` replaces Part III entirely while leaving other Parts untouched.

**FR-05.3 Auto-fill immutability** — auto-fill creates a new snapshot profile. The source profile is never mutated. `snapshot_profile_id` is returned so the user can compare/diff.

**FR-05.4 Agent contract** — request to agent includes `{espd_data, opportunity_id, company_id}`. Response yields `{espd_data, changed_fields[], opportunity_id, source_profile_id}`. `X-Caller-Service: client-api` header mandatory. Timeout 30s; no client-side retry (AI Gateway handles retry/circuit-breaker).

**FR-05.5 Agent error handling** — 503 or timeout from AI Gateway returns 503 to caller with `{"message": "AI features are temporarily unavailable…", "code": "AGENT_UNAVAILABLE"}`. No snapshot profile created on failure.

**FR-05.6 XML export schema** — root `<ESPDResponse xmlns="urn:X-eusolicit:espd:schema:v1">`, child `<PartII>`, `<PartIII>`, `<PartIV>`, `<PartV>` elements. UTF-8. Absent/empty Parts omitted. Must be well-formed (ElementTree-parseable).

**FR-05.7 PDF export** — uses `reportlab`. Header with company name (from `part_ii.operator_name` fallback "Unknown Operator"), title, profile name, generation date. Each Part rendered as labeled section.

**FR-05.8 Multiple profiles** — a company may hold multiple ESPD profiles (e.g., "Default," "Joint Venture with X"). No hard limit; pagination not required.

### NFRs

- **NFR-05.1** Auto-fill round-trip P95 < 10s including AI Gateway call.
- **NFR-05.2** XML/PDF export synchronous for profiles ≤ 2MB `espd_data`; stream larger via chunked response.
- **NFR-05.3** ESPD XML passes schematron validation against EU ESPD Response schema (offline validation, not enforced by API).

### Negative paths

- 404: cross-tenant or missing profile.
- 422: missing `profile_name`, invalid `format`, invalid `espd_data` shape.
- 503: AI Gateway unavailable.

### Watchlist / scope-extension proposals

**NS-05.1** — **Schematron validation** — the exhaustive pass revealed no validation of generated XML against the official EU ESPD schema. Propose offline schematron file + optional `?validate=true` on export that returns 422 with validation errors. **S13.31 — ESPD XML schematron validation**.

**NS-05.2** — **Multi-entity profiles for consortia** — a single profile holds one operator. Consortium bids need coordinated ESPD across multiple legal entities. Propose `espd_consortium_profiles` model with member legal-entity references. **S13.32 — Consortium ESPD multi-entity support**.

**NS-05.3** — **ESPD Part IV–VI granularity** — Part IV (selection criteria) has 15+ sub-criteria types (economic turnover, specific turnover per sector, professional risk insurance, etc.). The current JSONB is free-form. Propose structured validators per sub-criterion for better agent accuracy. **S13.33 — Structured ESPD Part IV sub-criterion validators**.

**NS-05.4** — **AI-assist on free-text fields** — some Part III questions require free-text explanations (e.g., "describe any breach of obligations in the last 3 years"). Propose AI assist for drafting these. **S13.34 — Free-text AI assistance for ESPD narratives**.

**NS-05.5** — **Version history** — profiles today are mutable with no history. Regulatory audits may require reconstructing the profile state at the time of bid submission. Propose `espd_profile_versions` append-only table capturing snapshots on `submit`. **S13.35 — ESPD profile version history for regulatory audit**.

---

## GAP-08 — Per-Bid Add-On Pricing

### Rationale (strategic)
Paid tiers get add-ons for premium AI features without full subscription upgrade. Monetizes power-usage without tier churn. Coverage is ✅ delivered via story 8-9.

### Data model

**DM-08.1** `client.add_on_purchases`:
- `id` UUID PK
- `company_id` UUID FK
- `opportunity_id` UUID FK (scope of the purchase)
- `add_on_type` ENUM {proposal_generation, deep_compliance_audit, pricing_analysis}
- `stripe_checkout_session_id` VARCHAR UNIQUE (idempotency key)
- `user_id` UUID (who initiated)
- `purchased_at` TIMESTAMPTZ default now
- Index `(company_id, opportunity_id, add_on_type)` for status query hot path

### API contracts

**API-08.1** `POST /api/v1/billing/addon/checkout/session` — body `{opportunity_id, add_on_type}`. Role: bid_manager / admin. Returns `{checkout_url, session_id}`. Stripe session `mode=payment` with metadata `{type: "addon", company_id, opportunity_id, add_on_type, user_id}`.

**API-08.2** `GET /api/v1/billing/addon/status?opportunity_id=X&add_on_type=Y` — any authenticated user. Returns `{purchased: bool, purchased_at: ISO8601 | null, add_on_type}`.

**API-08.3 Webhook handler** (existing `/api/v1/billing/stripe/webhook`)
- Branch for `checkout.session.completed` where `session.mode=payment AND metadata.type=addon`:
  1. Deduplicate via `webhook_events`.
  2. Extract metadata.
  3. `INSERT ... ON CONFLICT (stripe_checkout_session_id) DO NOTHING RETURNING id`.
  4. Audit `action_type=billing.addon_purchased`.
  5. Return `{status: ok, action: addon_purchase_recorded}`.
- Subscription `mode=subscription` sessions explicitly ignored with `{status: ok, action: checkout_session_ignored}`.

### Functional requirements

**FR-08.1 Add-on types** — three canonical types: `proposal_generation`, `deep_compliance_audit`, `pricing_analysis`. Price IDs in env (`STRIPE_PRICE_ADDON_PROPOSAL_GENERATION`, etc.) — NEVER hardcoded.

**FR-08.2 Unconfigured price** — if requested type has no configured Stripe price ID, return 422 `{"error": "addon_not_configured"}`.

**FR-08.3 Opportunity scoping** — add-on unlocks the feature for that opportunity only. Purchasing for Opportunity A does NOT unlock for Opportunity B.

**FR-08.4 Usage non-counting** — add-on consumption does NOT count against monthly subscription limits.

**FR-08.5 Idempotent purchase** — webhook `checkout.session.completed` is idempotent via `stripe_checkout_session_id` unique constraint + `ON CONFLICT DO NOTHING`.

**FR-08.6 Tax** — Stripe Tax enabled via `automatic_tax={"enabled": true}` on the checkout session (see GAP-09).

**FR-08.7 Frontend success redirect** — Stripe return URL includes `?addon_purchased=true`. Frontend:
- Invalidates addon-status query.
- Shows `toast.success(t('billing.addonPurchaseSuccess'))`.
- Strips the query param via `router.replace()` (no history pollution).
- Re-renders the opportunity detail with "Unlocked" badge for that add-on.

### NFRs

- **NFR-08.1** Webhook idempotency rigorous: double delivery inserts once; status query reflects single purchase.
- **NFR-08.2** Status query P95 < 50ms via `(company_id, opportunity_id, add_on_type)` index.

### Negative paths

- 422 unconfigured add-on type.
- 403 for non-bid_manager/admin initiating checkout.
- 502 on Stripe API failure at checkout session creation.

### Watchlist / scope-extension proposals

**NS-08.1** — **Refund policy + mechanics**. No documented refund flow. An accidental purchase today is permanent. Propose: admin-api endpoint to refund via Stripe + update `add_on_purchases.refunded_at`. **S13.36 — Add-on refund admin flow**.

**NS-08.2** — **Time-limited add-ons / expiry**. Add-ons are "forever" today per opportunity. Proposal-generation for a tender whose deadline was 6 months ago has no value. Propose optional `expires_at` column + background cleanup. **S13.37 — Add-on expiry policy**.

**NS-08.3** — **Bundle pricing** — e.g., "full-proposal-workflow bundle" (proposal generation + compliance audit + pricing) at a discount. Requires new product definitions in Stripe + a `bundle_id` column. **S13.38 — Add-on bundles**.

**NS-08.4** — **Add-on sharing across collaborators**. An add-on purchased by user A is accessible to all company members on that opportunity (current behavior — confirm). If not, surface on the purchase UI. Verification story: **S13.39 — Verify add-on visibility for all proposal collaborators**.

**NS-08.5** — **Invoice/receipt issuance** — Stripe issues receipts by default. Confirm they match EU VAT requirements (see GAP-09). If not, generate supplementary PDF. **S13.40 — Add-on VAT-compliant receipt (if Stripe default insufficient)**.

---

## GAP-09 — EU VAT Handling

### Rationale (strategic)
EU VAT compliance is legally mandatory for a SaaS selling to EU businesses. Reverse charge, VIES validation, custom Enterprise invoicing. Coverage is ✅ delivered via stories 8-10, 8-11.

### Data model

**DM-09.1** `client.companies.vat_validation_status` VARCHAR(20) NULL default 'pending'. Values: `pending | valid | invalid`. NULL for pre-existing rows (no validation attempted).

**DM-09.2** `client.enterprise_invoices`:
- `id` UUID PK
- `company_id` UUID FK ON DELETE RESTRICT (historical invoices must not vanish with company delete)
- `stripe_invoice_id` VARCHAR(255) UNIQUE
- `stripe_customer_id` VARCHAR(255)
- `admin_id` UUID (platform_admin who issued; NOT FK to client.users)
- `payment_terms` VARCHAR(10) CHECK IN (`NET_30`, `NET_60`)
- `line_items` JSONB NOT NULL default `[]`
- `status` VARCHAR(20) default `open` {open, paid, void}
- `created_at`, `sent_at`, `paid_at` TIMESTAMPTZ

### API contracts

**API-09.1 Stripe Tax everywhere** — `automatic_tax={"enabled": True}` on every `stripe.checkout.Session.create()` call (subscription + add-on).

**API-09.2 VIES validation** (`client-api`)
- `POST /api/v1/billing/vat/validate` — body `{vat_number: str}`. Any role. Validates format → calls VIES REST API with 3 retries exponential backoff 1s/2s/4s ±25% jitter → returns `{status, country_code, name, validated_at}`.
- Persists `companies.tax_id + vat_validation_status` on success.

**API-09.3 Enterprise invoicing** (`admin-api`)
- `POST /api/v1/admin/enterprise-invoicing/` — platform_admin. Body `{company_id, payment_terms: NET_30|NET_60, line_items[], memo?}`. LineItem: `{description: str (1..500), amount_cents: int ≥ 1, currency: str default "eur"}`. Max 50 line items.
- Steps: verify company.tier=enterprise → verify stripe_customer_id → create Stripe Invoice (draft) → add InvoiceItems → finalize → send → persist row (status=open, sent_at=now) → return 201 `EnterpriseInvoiceResponse`.
- 422 if company ≠ enterprise tier; 422 if no stripe_customer_id; 404 if company missing; 502 on Stripe error.

### Functional requirements

**FR-09.1 VIES contract** — format pre-validation first: first 2 chars ∈ EU country codes (ISO 3166-1 alpha-2), remainder national number. Malformed → `status="invalid"` without calling VIES.

**FR-09.2 VIES resilience** — VIES 503/timeout/connection error after 3 retries → `status="pending"`, NEVER raises to caller. Caller decides handling (typically keeps prior status).

**FR-09.3 Reverse charge** — Stripe Tax applies EU reverse charge automatically based on customer's country and business status. Invoice body notes "Reverse charge — Article 196 Council Directive 2006/112/EC" for applicable B2B cross-border charges.

**FR-09.4 Tax ID sync** — `companies.tax_id` updates sync to Stripe `customer.tax_ids` in a background task; failure logs WARN, does not block user request.

**FR-09.5 Enterprise invoice atomic flow** — Stripe Invoice create → add line items → finalize → send, all via `asyncio.to_thread()`. Failure at any step rolls back the Python transaction (local `enterprise_invoices` row is NOT created) but Stripe side-effects may persist — compensating cleanup is manual (flag in S13.41).

**FR-09.6 Invoice status reconciliation** — on `invoice.paid` and `invoice.voided` Stripe webhooks, update `enterprise_invoices.status` and `paid_at`. Idempotent via `webhook_events` dedup.

**FR-09.7 Tax ID on registration** — collected at Stripe Checkout or during profile completion; optional at registration for B2C, required for B2B to unlock reverse charge.

**FR-09.8 Invoice requirements** — every Stripe-generated invoice includes: seller VAT ID, buyer VAT ID (if provided), applicable VAT rate, net/gross amounts, legal basis for exemption.

### NFRs

- **NFR-09.1** VIES call P95 < 5s including 3 retries under error.
- **NFR-09.2** Enterprise invoice endpoint P95 < 8s including 4 Stripe round-trips.

### Negative paths

- 422 invalid VAT format (pre-VIES).
- VIES pending: 200 returned but `status=pending` — UI should re-try later or allow continue.
- 502 on Stripe API failure during enterprise invoice flow (local row not persisted).

### Watchlist / scope-extension proposals

**NS-09.1** — **Compensating cleanup for enterprise-invoice partial failures**. If Stripe Invoice is finalized but persisting the local row fails, Stripe state and our DB diverge. Propose: Celery reconciliation task that finds Stripe Invoices with no matching `enterprise_invoices` row and backfills. **S13.41 — Enterprise-invoice reconciliation task**.

**NS-09.2** — **Non-EU customer taxation** — US / APAC / UK post-Brexit need different handling. Stripe Tax covers most, but confirm coverage and document edge cases. **S13.42 — Non-EU tax handling documentation + any Stripe Tax config gaps**.

**NS-09.3** — **Tax-ID mid-cycle change** — if a company provides a tax ID mid-subscription, should past invoices be retroactively reverse-charged? Propose no retroactive adjustment, document in UX copy. **S13.43 — Tax-ID update UX + non-retroactive disclosure**.

**NS-09.4** — **VAT exemption flags** — some non-profits, municipalities, or charities are VAT-exempt. Stripe Tax has customer-level tax exempt flag; surface in admin-api. **S13.44 — VAT exemption admin toggle for qualifying customers**.

**NS-09.5** — **Credit notes on refunds** — Stripe supports credit notes via API. Issuing refund should auto-generate credit note PDF. **S13.45 — Automatic credit-note issuance on refunds**.

---

## GAP-11 — Per-Tender RBAC Granularity

### Rationale (strategic)
Users must be bid_manager on Tender A but read_only on Tender B. Without entity-level grants, the 5-role model at company level is too coarse. Coverage is ✅ delivered via stories 2-10 (generic entity RBAC) + 10-2 (proposal-level).

### Data model

**DM-11.1** `client.entity_permissions` (already per gap closure):
- `id` UUID PK
- `user_id`, `company_id` UUID
- `entity_type` VARCHAR (e.g., `"proposal"`, `"opportunity"`)
- `entity_id` UUID
- `permission` ENUM {read, write, manage}
- `granted_by` UUID; `granted_at`, `updated_at` TIMESTAMPTZ
- UNIQUE `(user_id, company_id, entity_type, entity_id, permission)` — prevents MultipleResultsFound 500s
- Composite IX `(user_id, entity_type, entity_id)` — hot path for the dependency

### Functional requirements — generic `check_entity_access` dependency

**FR-11.1 Default deny** — no `entity_permissions` row → access denied (explicit-grant model), unless role-bypass applies.

**FR-11.2 Role ceiling** — the caller's company role places an upper bound on what entity_permissions can grant:
- `admin` → any permission (bypass)
- `bid_manager` → up to `manage`
- `contributor` → up to `write` (even if row says `manage`)
- `reviewer` → up to `read`
- `read_only` → up to `read`

**FR-11.3 Admin/bid_manager bypass** — admin and bid_manager on the caller's own company bypass `entity_permissions` lookup entirely for entities owned by that company. Cross-company is 404.

**FR-11.4 Single optimized query** — the dependency performs the check in one SQL round-trip via the composite index. No N+1.

**FR-11.5 Per-request cache** — `request.state._rbac_cache[(entity_type, id, perm)] = bool`. A dependency called twice in the same request hits the cache.

**FR-11.6 Denial audit** — every 403 writes `shared.audit_log` with `action_type=access_denied`, `entity_type`, `entity_id`, `user_id`, `ip_address`. Audit write uses a dedicated session (survives the denial rollback) and MUST NOT block the response on transient audit failure.

### Functional requirements — proposal-level `require_proposal_role`

**FR-11.7 Factory signature** — `require_proposal_role(*allowed_roles, proposal_id_param="proposal_id")` returns an async FastAPI dependency.

**FR-11.8 Resolution order**:
1. Extract `proposal_id` from `request.path_params[proposal_id_param]`; 422 if not UUID-coercible.
2. **Admin bypass**: `current_user.role == CompanyRole.admin` → return immediately, no DB hit.
3. Cache lookup `request.state._rbac_cache[("proposal_role", str(proposal_id), frozenset(allowed_roles))]` — hit returns cached result.
4. Tenant check: load proposal; `company_id != current_user.company_id` → 404 (existence-leakage safe).
5. Collaborator lookup: row missing → 403 + denial audit.
6. Role check: `role not in allowed_roles` → 403 with `detail` listing held role + allowed set + denial audit.
7. Cache positive result and return.

**FR-11.9 Canonical role sets** — `core/rbac.py` exports:
- `PROPOSAL_ALL_ROLES` — all 5 (for read-only collaborator views).
- `PROPOSAL_WRITE_ROLES` — all 4 except read_only (for content writes per Epic 10 AC13).
- `PROPOSAL_BID_MANAGER_ONLY` — bid_manager only (destructive ops: delete, collaborator mgmt).

**FR-11.10 Cache coexistence** — uses `dict.setdefault` on `request.state._rbac_cache` so it coexists with `check_entity_access`'s `(entity_type, id, perm)` keys via the `"proposal_role"` key namespace.

### NFRs

- **NFR-11.1** Cache hit path adds <1ms; cache miss adds one indexed SELECT.
- **NFR-11.2** Denial audit failure MUST not convert 403 into 500.

### Negative paths

- 404 on cross-tenant proposal (existence-leakage).
- 403 on missing collaborator row or disallowed role — both audit as `access_denied`.
- 422 on path parameter coercion failure.

### Watchlist / scope-extension proposals

**NS-11.1** — **Opportunity-level grants**. `entity_permissions` supports any `entity_type`, but there is no product surface today to grant opportunity-level permissions (e.g., "read_only on this opportunity while blocked from editing"). The gap doc mentioned the need. Propose: UI + API to grant opportunity-level `entity_permissions`. **S13.46 — Opportunity-level permission grants**.

**NS-11.2** — **Bulk revocation on user deactivation / leave**. When a user is deactivated (via story 2-9), their entity_permissions rows remain — stale access if the account is reactivated. Propose: on deactivation, soft-revoke all entity_permissions; on reactivation, surface admin review UI. **S13.47 — User-deactivation permission revocation**.

**NS-11.3** — **Inherited / template permissions**. Granting 5 permissions to a new collaborator one at a time is tedious. Propose: permission templates ("Technical Reviewer template = read on opportunity + write on proposal sections X, Y"). **S13.48 — Permission templates**.

**NS-11.4** — **Permission audit view** — a user's total accessible entities across both role + entity grants. Useful for compliance export. **S13.49 — Per-user permission audit view (admin)**.

---

## GAP-12 — Reusable Content Blocks Library

### Rationale (strategic)
Boilerplate sections (company overview, quality management, sustainability policy) are reused verbatim or near-verbatim across proposals. A curated, approved library reduces drafting time and ensures brand consistency. Coverage is ✅ delivered via stories 7-9, 7-16.

### Data model

**DM-12.1** `client.content_blocks`:
- `id` UUID PK
- `company_id` UUID FK (RLS derived from JWT)
- `title` VARCHAR (required)
- `category` VARCHAR (e.g., "company_overview", "quality_management", "sustainability", "case_studies")
- `body` TEXT (required)
- `tags` TEXT[] — array of tag strings
- `version` INT default 1 (increments on PATCH)
- `approved_at` TIMESTAMPTZ NULL
- `approved_by` UUID NULL
- `search_vector` TSVECTOR — full-text search over title + body, refreshed in service layer (not relying on DB trigger per E07-R-009 mitigation)
- `created_at`, `updated_at` TIMESTAMPTZ

### API contracts

**API-12.1** (`/api/v1/content-blocks`)
- POST — body `{title, category, body, tags[]}`. `company_id` from JWT. `version=1`, `approved_at=null`. 201.
- GET — list with filters: `category` (exact), `tags` (CSV, AND-semantics via `@>`), pagination `limit` (≤100, default 20), `offset` (default 0). Default sort `created_at DESC`.
- **GET /search?q=…** — MUST be registered before `/{block_id}` in router (FastAPI path-collision pitfall). Uses `plainto_tsquery('english', q)` on `search_vector`. Sorted by `ts_rank DESC`. Cross-company returns empty.
- GET /{id} — 404 cross-tenant/missing (no 403, no enumeration).
- PATCH /{id} — partial update; `version++`, `updated_at=now()`. If `title` or `body` in payload, recalculate `search_vector` in service explicitly.
- POST /{id}/approve — `approved_at=now(), approved_by=current_user.user_id`. 200.
- POST /{id}/unapprove — clears both approval fields. 200.
- DELETE /{id} — **hard-delete** (NOT soft-archive; unlike proposals). 204.

### Functional requirements

**FR-12.1 Tenant isolation** — `company_id` derived from JWT. Cross-tenant GET returns 0 rows / 404. PATCH/DELETE return 404 on cross-tenant.

**FR-12.2 Approval lifecycle** — blocks start unapproved; approve/unapprove transitions writable by any authorized user (verify RBAC: current ACs show no explicit role gate — may need review as NS-12.1).

**FR-12.3 Tag semantics** — `tags` filter uses AND: `tags=quality,EU` returns blocks tagged with BOTH. Empty tag filter returns all.

**FR-12.4 Full-text search refresh** — on PATCH that touches `title` or `body`, the service computes `to_tsvector('english', coalesce(new_title,'') || ' ' || coalesce(new_body,''))` and writes `search_vector` explicitly. No reliance on DB trigger.

**FR-12.5 Versioning** — `version` increments on each PATCH. No version history stored today; the integer is advisory.

**FR-12.6 RAG integration** — approved content blocks are fed into the Proposal Generator Workflow via KraftData Company Profiles Store. The content_blocks table is the source of truth; KraftData syncs via agent-side pull (stories 7-5, 7-9).

### NFRs

- **NFR-12.1** Search P95 < 250ms for libraries up to 1000 blocks with GIN index on `search_vector`.
- **NFR-12.2** List with tags filter P95 < 200ms with GIN index on `tags`.

### Negative paths

- 404 cross-tenant/missing block on GET /{id}, PATCH, DELETE.
- 422 empty title or body on POST.

### Watchlist / scope-extension proposals

**NS-12.1** — **Approval authorization gate**. The current AC does not specify who can approve a block. Propose: only bid_manager or admin can approve/unapprove. **S13.50 — Content-block approval RBAC gate**.

**NS-12.2** — **Usage analytics per block** — "most-inserted" is a useful metric but not tracked today. Propose: `content_block_usages` table (block_id, proposal_id, inserted_by, inserted_at) written on insertion in the editor. **S13.51 — Content-block usage analytics**.

**NS-12.3** — **Block inheritance / parent-child variants** — a parent "Company Overview" block with child variants "Company Overview (EU Grant)" and "Company Overview (Public Tender)" that override specific paragraphs. Reduces duplication. Complex — design spike first. **S13.52 — Block inheritance design spike**.

**NS-12.4** — **Block archive / deprecation** — DELETE today is hard. Archive flag would preserve historical proposal references. Propose: add `archived_at` column, DELETE becomes soft, hard-delete migrates to an admin endpoint with warning. **S13.53 — Content-block soft-archive**.

**NS-12.5** — **Translation alignment (BG/EN pairs)** — blocks today are single-language. For a BG/EN platform with i18n, pairing translated blocks is valuable. Propose: `translation_group_id` column linking BG/EN variants. **S13.54 — BG/EN content-block translation alignment**.

**NS-12.6** — **Block version history** — `version` is advisory integer. No audit of what changed and when. Propose: `content_block_versions` append-only snapshot on each PATCH. **S13.55 — Content-block version history audit**.

---

## GAP-13 — Bid History Analytics & Custom Reporting

### Rationale (strategic)
Analytics dashboards + scheduled reports are the feedback loop for improving win rate and proving platform ROI. Without them the platform cannot demonstrate value. Coverage is ✅ delivered via a large set of stories: 10-11 (bid outcomes + prep logs), 12-1 (MV infrastructure), 12-2/12-3 (market intel), 12-4 (ROI), 12-5 (team performance), 12-6 (competitor intel), 12-7 (pipeline forecast), 12-8 (usage dashboard), 12-9 (PDF/DOCX report generator), 12-10 (scheduled delivery), 9-14 (Celery task).

### Data model

**DM-13.1 Existing analytics-source tables** (post Migration 045):
- `client.bid_outcomes` — outcome per proposal: status (won/lost/withdrawn), contract_value, evaluator_feedback, evaluator_scores, lessons_learned (AI-filled), recorded_by, company_id, opportunity_id, note.
- `client.bid_preparation_logs` — time/cost ledger: proposal_id, user_id, activity_type (drafting/review/research/meeting/other), hours (decimal), cost_eur (decimal nullable), logged_at, notes, company_id (denormalized), opportunity_id (denormalized).

**DM-13.2 Materialized views** (created in Migration 011, refreshed by Celery):
- `mv_market_intelligence` — sector-level aggregates (procurement volumes, avg contract values, active authorities, seasonal trends).
- `mv_roi_tracker` — per-company per-month invested / won / ROI.
- `mv_team_performance` — per-user bids_submitted, win_rate, avg_preparation_time.
- `mv_competitor_intelligence` — aggregated competitor patterns (Professional tier only).
- `mv_usage_consumption` — per-company usage vs. tier limits, hourly.

Each MV has a UNIQUE index on its natural key (required for `REFRESH MATERIALIZED VIEW CONCURRENTLY`).

### API contracts

**API-13.1 Dashboard endpoints** (`/api/v1/analytics/…`, Professional+)
- `GET /market` — `{sector_aggregates[], avg_contract_values, trends}`, filters: `date_from`, `date_to`, `cpv_prefix`.
- `GET /roi/summary` — `{total_invested_eur, total_won_eur, roi_pct, bid_count}`; filters `date_from`, `date_to`. Returns zeros (not 404) when no data.
- `GET /roi/bids` — paginated per-bid `{proposal_id, month, total_invested_eur, total_won_eur, roi_pct}`. `{items, total, page, page_size}`.
- `GET /roi/monthly` — time-series for chart.
- `GET /team` — per-user aggregates, filters: `user_id`, `date_range`.
- `GET /competitor` — competitor win/price benchmarks (Prof+).
- `GET /forecast` — predicted opportunities (Prof+).
- `GET /usage` — current-period usage vs. tier limits. `All paid`.

**API-13.2 Report generation** (`/api/v1/reports/…`, Professional+)
- POST `/generate` — body `{report_type, params}`. Returns `{job_id}`. Asynchronous Celery job.
- GET `/jobs/{job_id}` — `{status, artifact_url?, error?}`.
- GET `/list` — list of past reports for the company.
- DELETE `/{id}` — remove report artifact (soft).

### Functional requirements

**FR-13.1 Preparation log write path** — writes captured automatically (UI surface exists for manual logging via stories 10-11/12-4/12-5) OR via proposal-editor session telemetry. Covers both explicit and inferred logs.

**FR-13.2 Outcome recording** — `POST /api/v1/proposals/{id}/outcomes` with body `{status, contract_value?, evaluator_feedback?, evaluator_scores?, note?}`. Role: bid_manager/admin.

**FR-13.3 Lessons Learned agent trigger** — on `bid_outcome_recorded` event, Notification Service triggers the Lessons Learned agent via AI Gateway; response persisted in `bid_outcomes.lessons_learned` JSONB.

**FR-13.4 MV concurrent refresh** — all refreshes use `REFRESH MATERIALIZED VIEW CONCURRENTLY`. Dashboard reads never block during refresh.

**FR-13.5 Refresh schedule** — daily 02:00–02:45 AM UTC staggered for market/roi/team/competitor; hourly `:00` for usage.

**FR-13.6 Report types** — MVP set: `pipeline_value`, `success_rate_trends`, `upcoming_deadlines`, `roi_report`, `team_performance`. Extensible via agent registration for custom narrative reports.

**FR-13.7 Report delivery** — async Celery job generates PDF/DOCX → stores in S3 → publishes `report.generated` event → Notification Service emails the requester with the signed URL.

**FR-13.8 Scheduled reports** — user configures `report_schedules(user_id, report_type, params, cadence)`. Cron-style cadence (daily/weekly/monthly). Runs at 08:00 local time.

**FR-13.9 Tier gating**:
- Free: no analytics.
- Starter: usage dashboard only.
- Professional: market, ROI, team, forecast (not competitor).
- Enterprise: all dashboards + custom report builder.

**FR-13.10 GDPR filter** — team performance dashboard must redact individual user names for users who have set `gdpr_analytics_opt_out=true` on their profile. Aggregate totals preserved, attribution anonymized.

### NFRs

- **NFR-13.1** Dashboard endpoints P95 < 500ms (MV-backed).
- **NFR-13.2** Report generation async — response within 5s even for large data sets.
- **NFR-13.3** MV refresh completes within 30 minutes for all 5 views at 1M+ rows scale.
- **NFR-13.4** Report artifacts retained 90 days by default; lifecycle rule on S3 bucket.

### Negative paths

- 404 cross-tenant proposal / outcome.
- 403 on tier-gated analytics endpoints below required tier.
- 409 on duplicate outcome record (one `BidOutcome` per proposal).

### Watchlist / scope-extension proposals

**NS-13.1** — **Custom report builder** — user-defined metric combinations (e.g., "bids with evaluator_score > 80 AND status=lost") beyond the fixed set. Requires a constrained DSL or drag-and-drop UI. Complex. **S13.56 — Custom report builder (Enterprise)**.

**NS-13.2** — **Export scheduling granularity** — today cadence is daily/weekly/monthly at 08:00. Full cron expression support for Enterprise use cases. **S13.57 — Cron-style report scheduling**.

**NS-13.3** — **Dashboard drill-down** — from an aggregated metric (e.g., "avg preparation time 48h") drill to the rows contributing. Requires per-MV detail endpoints. **S13.58 — Dashboard drill-down to source rows**.

**NS-13.4** — **Benchmark comparisons across companies** — anonymized cross-company benchmarks ("your win rate vs. industry median"). Privacy-sensitive. Requires opt-in and k-anonymity. **S13.59 — Anonymized cross-company benchmarks**.

**NS-13.5** — **Real-time vs. MV trade-off** — usage dashboard refreshes hourly; some UI expectations are "live." Propose: usage endpoint queries source tables directly (no MV) for current period; MV used only for historical periods. **S13.60 — Live-current-period usage query path**.

**NS-13.6** — **Report template versioning** — generated reports don't track template version. If template changes, old reports should remain reproducible OR clearly labeled "generated with template v2". **S13.61 — Report template versioning + reproducibility**.

**NS-13.7** — **Report artifact retention config** — today 90 days hardcoded. Enterprise may need 7-year retention (EU audit). Propose per-company retention policy. **S13.62 — Per-company report retention policy**.

---

## GAP-14 — KraftData Agent Inventory Reconciliation 📝

### Current state
- Solution Architecture v4 §11.2 lists **29 agents** (reconciled).
- Requirements Brief v4 §7 lists **22 agents** (pre-reconciliation).
- Delta: Consortium Finder, Reporting Template Generator, ESPD Auto-Fill, Submission Guide, Lessons Learned, Win Theme Extractor, Requirement Checklist (the 7 additions).

### Functional requirements

**FR-14.1** Requirements Brief v5 §7 MUST match Architecture v4 §11.2 verbatim in count (29 entries) and content.

**FR-14.2** Each agent row MUST carry: `#`, `name`, `type (Agent | Team | Workflow)`, `ownership (KraftData | Local)`, `consumer_service`. The v4 §11.2 table already uses this schema.

**FR-14.3** **New entries in v5 Requirements Brief §7** (added vs. v4):
- #14 Consortium Finder Agent
- #15 Reporting Template Generator Agent
- #18 Requirement Checklist Agent
- #19 Win Theme Extractor Agent
- #26 ESPD Auto-Fill Agent
- #27 Submission Guide Agent
- #28 Lessons Learned Agent

**FR-14.4** **Entries removed from v5 Requirements Brief §7** (they were crawlers — see GAP-15):
- AOP Crawler Agent (KraftData listing)
- TED Crawler Agent (KraftData listing)
- EU Grant Portal Agent (KraftData listing)

### No new stories; this is a doc edit consumed by the Requirements Brief v5 pass.

---

## GAP-15 — Crawler Implementation Strategy Documentation 📝

### Current state
- Architecture v4: Crawlers are **local Celery tasks** in the `data-pipeline` service (stories 5-4 AOP, 5-5 TED, 5-6 EU Grants).
- Requirements Brief v4: describes crawlers as **KraftData agents**, contradicting the build.

### Functional requirements (documentation)

**FR-15.1** Requirements Brief v5 §5.2 Data Pipeline Architecture MUST state:
> Crawlers are deterministic HTTP scrapers implemented as Celery tasks inside the `data-pipeline` service. They handle portal-specific HTML/XML parsing for AOP (Bulgaria public procurement), TED (EU public procurement), and the EU Funding & Tenders Portal. KraftData agents are invoked downstream of crawling for normalization, enrichment, relevance scoring, and generation tasks — not for the crawling itself.

**FR-15.2** Requirements Brief v5 §7 (Agent Inventory) MUST NOT list crawlers as KraftData agents. Crawlers are acknowledged in §5.2 as local.

**FR-15.3** PRD v2 §3.1 "Multi-source monitoring" AC row should be clarified: "AOP, TED, and EU grant portals crawled by local Celery tasks on schedule; new opportunities appear within 24h of publication."

### Rationale for the architectural choice (to document)
- Crawlers are deterministic HTTP scraping, not AI reasoning. Running them locally gives us full control over error handling, rate limiting, and portal-specific quirks.
- KraftData agent cost model is per-call; paying per-crawl for 10K+ opportunities/day would be uneconomical.
- Portal authentication, cookies, and session state are best handled in trusted local code.

### No new stories; this is a doc edit.

---

## Summary of proposed new stories from P2 exhaustive pass

| Story | Gap | Title | Priority |
|-------|-----|-------|----------|
| S13.31 | GAP-05 | ESPD XML schematron validation | P2 (correctness) |
| S13.32 | GAP-05 | Consortium ESPD multi-entity support | P3 (post-MVP candidate) |
| S13.33 | GAP-05 | Structured ESPD Part IV sub-criterion validators | P3 (agent accuracy) |
| S13.34 | GAP-05 | Free-text AI assistance for ESPD narratives | P2 (UX) |
| S13.35 | GAP-05 | ESPD profile version history for regulatory audit | P2 (compliance) |
| S13.36 | GAP-08 | Add-on refund admin flow | P2 (ops) |
| S13.37 | GAP-08 | Add-on expiry policy | P2 (ops) |
| S13.38 | GAP-08 | Add-on bundles | P3 (commercial) |
| S13.39 | GAP-08 | Verify add-on collaborator visibility | P3 (verification) |
| S13.40 | GAP-08 | Add-on VAT-compliant receipt (conditional) | P3 (verify-first) |
| S13.41 | GAP-09 | Enterprise-invoice reconciliation task | P2 (correctness) |
| S13.42 | GAP-09 | Non-EU tax handling documentation + config review | P2 (commercial) |
| S13.43 | GAP-09 | Tax-ID update UX + non-retroactive disclosure | P3 (polish) |
| S13.44 | GAP-09 | VAT exemption admin toggle | P2 (compliance) |
| S13.45 | GAP-09 | Automatic credit-note issuance on refunds | P2 (compliance) |
| S13.46 | GAP-11 | Opportunity-level permission grants | P2 (watchlist) |
| S13.47 | GAP-11 | User-deactivation permission revocation | P2 (security) |
| S13.48 | GAP-11 | Permission templates | P3 (UX) |
| S13.49 | GAP-11 | Per-user permission audit view | P3 (compliance export) |
| S13.50 | GAP-12 | Content-block approval RBAC gate | P2 (correctness) |
| S13.51 | GAP-12 | Content-block usage analytics | P3 (insight) |
| S13.52 | GAP-12 | Block inheritance design spike | P3 (spike) |
| S13.53 | GAP-12 | Content-block soft-archive | P2 (data hygiene) |
| S13.54 | GAP-12 | BG/EN content-block translation alignment | P2 (i18n) |
| S13.55 | GAP-12 | Content-block version history audit | P3 (audit) |
| S13.56 | GAP-13 | Custom report builder (Enterprise) | P3 (Enterprise-only) |
| S13.57 | GAP-13 | Cron-style report scheduling | P3 (Enterprise-only) |
| S13.58 | GAP-13 | Dashboard drill-down | P2 (UX) |
| S13.59 | GAP-13 | Anonymized cross-company benchmarks | P3 (design spike first) |
| S13.60 | GAP-13 | Live-current-period usage query path | P2 (UX) |
| S13.61 | GAP-13 | Report template versioning + reproducibility | P3 (audit) |
| S13.62 | GAP-13 | Per-company report retention policy | P3 (Enterprise-only) |

**Total new stories from P2**: 32, none gap-closing (P2 gaps were all delivered; these are watchlist-driven refinements).

**Zero new stories for GAP-14 / GAP-15** — both are documentation reconciliation, consumed by Requirements Brief v5 (FR-14.1 through FR-15.3).

---

## Running tally (P1 + P2)

- **Gap-closing stories**: 4 (all from GAP-06, S13.1–S13.4)
- **Watchlist-driven stories**: 53 (21 from P1 + 32 from P2)
- **Documentation-only deltas** (no stories): GAP-14 + GAP-15 in Requirements Brief v5
- **Total new stories**: **57**

Next: Task #4 — P3 (GAP-03 Consortium Finder, GAP-04 Post-Award deferred). I expect minimal story proposals since GAP-04 is explicitly out-of-MVP.
