# E15: Per-Bid SKU + Pro+ Tier Launch

**Sprint**: 16–17 | **Points**: 21 | **Dependencies**: E08, E14 | **Milestone**: Consulting-firm monetisation
**Source:** PRD v1.1 §6 FR2.4–2.7 + §4 (Pro+ tier); architecture-evaluation §2 Change-2, §11 locked decisions §11.2 and §11.3

## Goal

Launch the Pro+ tier (€199/user/month) and surface the Per-Bid SKU as a customer-facing pricing primitive. The Stripe one-time-payment infrastructure for per-bid add-ons already exists in the v5 architecture (`add_on_purchases` table + Stripe `mode: payment` flow per ADR §1.14, shipped in Epic 8) — this epic extends pricing tiers, integrates with Pro+ feature gating, and adds the metering-bypass logic so per-bid purchases unlock per-opportunity feature use without consuming subscription tier caps.

The Pro+ tier gates: multi-client workspace (Epic 14), CRM integrations (Epic 17), ISO 27001 posture, dedicated CSM, public outcome-API access (Epic 19). The per-bid SKU provides three pricing tiers — €99 standard / €199 with grant module / €299 full Enterprise feature stack — letting consulting firms pass cost through to client engagements without subscribing.

## Acceptance Criteria

- [ ] Pro+ tier added to `tier_access_policies` with feature flags (multi-client workspace = unlimited; CRM = enabled; outcome API = enabled; dedicated CSM = enabled)
- [ ] Stripe Price ID for Pro+ at €199/user/month created; tier-cache invalidation publishes `subscription.changed` event (existing Epic 8 pattern) on tier transitions
- [ ] `client.add_on_pricing_tiers` table created with three default rows (€99 / €199 / €299) and admin-configurable via Admin API
- [ ] Per-bid SKU UX surfaced on opportunity detail page with three-tier picker; defaults to standard (€99) tier
- [ ] Stripe Checkout (mode: payment) launches with selected per-bid tier price; on `checkout.session.completed` webhook, `add_on_purchases` row created with `add_on_pricing_tier_id` linkage and `opportunity_id` scoped (existing Epic 8 webhook handler extended)
- [ ] `_USAGE_LUA` Redis Lua script extended with pre-check: if `add_on_purchases` is active for `opportunity_id` in scope, bypass tier-level monthly cap and increment usage normally for analytics
- [ ] Concurrent INCR integration test (testcontainers Redis): 10 parallel calls to a metered feature on a per-bid-active opportunity all succeed without contention; tier cap remains untouched
- [ ] Refund policy: per-bid purchases non-refundable after 24-hour cancellation window (Stripe-standard, Epic 8 pattern reused)
- [ ] Per-bid activation triggers `addon.opportunity_engaged` event when `bid_decisions.recommendation == 'bid'` is logged for the opportunity
- [ ] EU VAT MOSS reconciliation: Stripe Tax product mapping for per-bid SKUs distinct from subscription SKUs; tax-reporting test covers both
- [ ] Pro+ pricing page renders accurate feature comparison vs Professional and Enterprise tiers
- [ ] TierGate Depends() pattern (Epic 6) extends to Pro+ without refactor — verified by source-inspection ATDD

## Stories

### S15.00: Pro+ Tier Definition + Stripe Price + Tier-Cache Extension
**Points**: 5 | **Type**: backend

Add `pro_plus` enum value to `subscription.tier` column (Alembic migration). Insert one row into `tier_access_policies` with all Pro+ feature flags. Create one Stripe Price ID at €199/user/month (recurring) via Stripe Dashboard or API; record in `infra/stripe-config.yaml`. Extend the tier-cache (existing Epic 8 pattern: DELETE-on-change followed by 60s repopulate) to handle `pro_plus` transitions. Publish `subscription.changed` event with `old_tier` and `new_tier` for cross-service cache invalidation. Cross-tenant negative test mandatory: customer A on Pro+ cannot access customer B's Pro+ resources.

---

### S15.01: Per-Bid Pricing Tiers + Stripe Checkout Integration + Admin Config
**Points**: 8 | **Type**: fullstack

Create `client.add_on_pricing_tiers` table with columns: `id`, `name` (standard/grant_module/enterprise_stack), `price_eur_cents`, `stripe_price_id`, `feature_stack` (JSONB enumerating which features unlock), `active`. Seed three default rows (€99, €199, €299) with corresponding Stripe Price IDs. Implement Admin API CRUD for tier management (admin-only).

Frontend: opportunity detail page renders a three-tier picker with feature-stack tooltips. On selection, frontend calls `POST /api/v1/opportunities/:id/per-bid/checkout` which creates a Stripe Checkout Session (mode: payment) with the selected tier's Price ID, success/cancel URLs, and metadata `{opportunity_id, workspace_id, tier_id}`. Frontend redirects to Checkout URL.

Webhook handler (Epic 8 extension): on `checkout.session.completed`, create `add_on_purchases` row with `opportunity_id`, `workspace_id`, `add_on_pricing_tier_id`, `stripe_payment_intent_id`, `status: succeeded`. Audit log entry written.

Cross-tenant negative test: customer A cannot purchase per-bid SKU on customer B's opportunity (403). Idempotency: webhook replays do not create duplicate purchases (existing Epic 8 dedup pattern).

---

### S15.02: _USAGE_LUA Metering Bypass + Concurrent INCR Integration Test
**Points**: 8 | **Type**: backend | **Atomic-Lua-critical**

Extend the existing `_USAGE_LUA` Redis Lua script (Epic 6 atomic counter — "GET + conditional INCR + EXPIRE in single script") with a per-bid bypass pre-check:

```lua
-- New pseudocode for the script:
-- 1. Check if KEYS[2] (per-bid active key for opportunity_id) exists
-- 2. If exists: INCR KEYS[1] (analytics counter only), return 0 (no tier cap check)
-- 3. Else: existing logic (GET tier cap → conditional INCR → return error if cap reached)
```

Per-bid active key format: `add_on_active:{workspace_id}:{opportunity_id}` set on `add_on_purchases.status=succeeded` with TTL = opportunity lifetime + 30 days. Set/delete fired by `add_on_purchases` insert/refund.

**Mandatory integration test (testcontainers Redis, NOT fakeredis — Epic 6 anti-pattern about concurrency):** spawn 10 parallel async tasks calling a metered feature on a per-bid-active opportunity; assert all succeed, tier cap counter remains untouched, analytics counter increments to 10, no race conditions or partial increments. Per project-context Epic 6 pattern: "Atomic Redis Lua script is the mandatory pattern for all metering operations."

Workspace-scoped negative test: per-bid activation on workspace W1's opportunity does not bypass tier caps for workspace W2 even within the same tenant.
