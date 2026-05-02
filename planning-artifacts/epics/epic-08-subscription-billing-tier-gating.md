# Epic 8: Subscription Billing & Tier Gating

Users can manage their Stripe-based subscriptions to unlock tiered features with strictly enforced usage limits.

### Story 8.1: Stripe Customer Provisioning
As a new user,
I want signup to be instant even if Stripe is slow,
So that I am not blocked from accessing the platform.

**Acceptance Criteria:**
**Given** a new registration
**When** the user is created
**Then** Stripe customer creation occurs asynchronously via BackgroundTask
**And** stripe_customer_id remains nullable until provisioned

### Story 8.2: Upgrade / Downgrade
As a Free user,
I want to upgrade in one click,
So that I can access premium features.

**Acceptance Criteria:**
**Given** I initiate a tier upgrade
**When** the Stripe checkout completes and webhook is received
**Then** webhook validation uses hmac.compare_digest()
**And** the tier cache is DELETEd and the new tier is enforced via Depends(TierGate)

### Story 8.3: Usage Metering
As an operator,
I want fair-use enforcement that is race-free,
So that users do not exceed their API limits.

**Acceptance Criteria:**
**Given** a user consumes AI gateway resources
**When** the usage is metered
**Then** an atomic Redis Lua script (_USAGE_LUA) increments and checks limits in one round trip
**And** billing endpoints gracefully handle Redis outages by failing open

### Story 8.4: Webhook Failure Paths
As an operator,
I want failed-payment events to drive tier changes,
So that unpaid accounts lose premium access.

**Acceptance Criteria:**
**Given** a Stripe invoice.payment_failed event
**When** the webhook is processed
**Then** the account tier transitions to past_due
**And** VIES VAT validation fails open to pending on timeout
