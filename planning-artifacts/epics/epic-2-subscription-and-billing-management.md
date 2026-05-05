---
epic: 2
title: Subscription & Billing Management
status: draft
source: eusolicit-docs/planning-artifacts/epics.md
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
frs_covered: ["FR-10", "FR-11", "FR-12", "FR-13", "FR-14"]
nfrs_relevant: ["NFR-1", "NFR-6", "NFR-7", "NFR-16"]
ux_drs_relevant: []
---

# Epic 2: Subscription & Billing Management

**Phase**: MVP | **Dependencies**: Epic 1 (Identity), Stripe integration in `client-api`

## Goal

Users can subscribe to paid tiers (Starter, Professional, Enterprise), experience a 14-day Professional free trial, manage their subscription via Stripe's customer portal, and purchase per-bid add-ons. The system must enforce tier-based feature gating across every other epic and process all financial mutations idempotently.

## User Outcome

After this epic, a registered user can pick a tier, pay with a card via Stripe Checkout, receive a 14-day Professional trial automatically on signup, switch tiers self-service, and unlock premium per-bid features without an enterprise contract.

## Epic Acceptance Criteria

- [ ] Stripe Checkout integration is wired into the client app and `client-api` for Starter, Professional, and Enterprise tiers.
- [ ] All Stripe webhook handlers are idempotent — duplicate webhook deliveries do not double-apply state changes (NFR-16).
- [ ] A `TierGate` dependency in FastAPI is the single source of truth for feature-gating decisions, returning 402 with a `paywall_required` error envelope when a tier is insufficient.
- [ ] Trial state, subscription state, and feature entitlements are derivable from `client.subscriptions` without requiring live calls to Stripe on each request.
- [ ] Stripe Tax is enabled so EU VAT is captured correctly per customer billing address.
- [ ] All checkout success/cancel pages are tenant-scoped — a logged-out URL replay cannot upgrade another tenant.

## Stories

### Story 2.1: Free Trial Auto-Enrollment

As a new user,
I want to be automatically enrolled in a 14-day free trial of the Professional tier,
So that I can evaluate the platform's advanced features risk-free.

**FR coverage**: FR-12

**Acceptance Criteria:**

**Given** I have just registered and verified my email
**When** I log in for the first time
**Then** a `client.subscriptions` row is created with `tier=professional`, `status=trialing`, `trial_ends_at=now()+14d`
**And** a trial countdown badge is visible in the top bar.
**And** at trial end, the subscription auto-transitions to `tier=free` (or to `professional` if a payment method has been provided).

---

### Story 2.2: Stripe Subscription Upgrade

As a user,
I want to upgrade my tier using my credit card via Stripe,
So that I can continue using premium features post-trial.

**FR coverage**: FR-10

**Acceptance Criteria:**

**Given** I am on the billing page
**When** I select a paid tier and complete Stripe Checkout
**Then** Stripe redirects me back to the success URL with a `session_id`
**And** a `customer.subscription.created` webhook updates `client.subscriptions` to `status=active` with the chosen tier
**And** retrying the same webhook event (same `event.id`) is a no-op due to the `stripe_events` idempotency table.

---

### Story 2.3: Feature Gating by Tier

As the platform,
I want to automatically restrict features based on the user's active tier and usage limits,
So that paywall rules are enforced and users are nudged to upgrade.

**FR coverage**: FR-11

**Acceptance Criteria:**

**Given** I am a user on a specific tier
**When** I attempt to access a tier-gated feature (e.g., AI tender analysis on Free)
**Then** the `TierGate` dependency checks my tier and usage counters
**And** allows or denies the call.
**And** denied requests return HTTP 402 with `{error_code: "paywall_required", required_tier, current_tier}` and the UI renders a contextual paywall modal with a "Compare plans" link.
**And** usage-quota gates (e.g., monthly AI tokens) increment counters atomically with feature use.

---

### Story 2.4: Subscription Self-Service Management

As a paying user,
I want to manage my subscription through Stripe's self-service portal,
So that I can upgrade, downgrade, or cancel without contacting support.

**FR coverage**: FR-13

**Acceptance Criteria:**

**Given** I have an active subscription
**When** I click "Manage billing"
**Then** the backend creates a Stripe Customer Portal session scoped to my Stripe customer
**And** I am redirected to Stripe.
**And** any changes I make (plan switch, cancel, payment method update) are reflected back in `client.subscriptions` via `customer.subscription.updated` webhook within 60s.

---

### Story 2.5: One-Time Add-On Purchases

As a user,
I want to purchase specific premium features on a per-bid basis,
So that I can access advanced capabilities for critical bids without committing to a higher tier.

**FR coverage**: FR-14

**Acceptance Criteria:**

**Given** I am viewing an opportunity
**When** I click "Unlock premium analysis" and complete Stripe Checkout (`mode=payment`)
**Then** a `client.addon_purchases` row is created tying the purchase to my `opportunity_id`
**And** the specific feature is unlocked for that opportunity only, not platform-wide.
**And** the purchase is itemised separately on my Stripe invoice (no subscription proration).

## Implementation Notes

- Idempotency is implemented in two layers: `stripe_events.event_id UNIQUE` table for webhook dedup, and `Idempotency-Key` header on outbound Stripe API calls.
- Webhook signature verification uses `stripe.Webhook.construct_event` with the configured signing secret — never `==` for HMAC compare.
- All financial mutations are wrapped in an audit-log emission via the shared event bus (Epic 8).
- Feature-flag-style gating data lives in `shared.tier_features` with effective-date support so we can A/B price changes.
- Frontend paywall modal uses a single `<UpgradePrompt>` component to keep messaging consistent.
