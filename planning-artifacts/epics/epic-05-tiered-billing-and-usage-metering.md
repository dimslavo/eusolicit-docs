# Epic 5: Tiered Billing & Usage Metering

Manage subscription tiers, enforce access limits, meter AI usage, and support the Per-Bid Pricing SKU.

### Story 5.1: Tier-Gated Subscriptions and Usage Metering

As a tenant admin,
I want my subscription tier to govern the features and AI usage limits for my firm,
So that I am billed fairly according to my scale and consumption.

**Acceptance Criteria:**

**Given** an active Stripe subscription for the tenant
**When** users across workspaces request AI features (summaries, drafts)
**Then** atomic Redis metering increments usage
**And** blocks requests with a 429 when the tenant's tier limits are exceeded.

### Story 5.2: Per-Bid Pricing SKU Purchasing

As a Bid Manager,
I want to purchase a Per-Bid SKU for a specific opportunity,
So that I can bypass monthly tier caps and pass the cost to the client engagement.

**Acceptance Criteria:**

**Given** I am viewing an opportunity
**When** I initiate a Per-Bid SKU purchase via Stripe one-time payment
**Then** the opportunity is flagged `per_bid_sku_active=true` upon successful payment
**And** all subsequent metered-feature usage for this opportunity bypasses the tier-level monthly caps.

### Story 5.3: In-Product NPS and Review Routing

As a Product Manager,
I want to prompt active Pro+ users for an NPS score quarterly,
So that I can capture feedback and route satisfied users to G2/Capterra.

**Acceptance Criteria:**

**Given** an active Pro+ tier user logging in
**When** the quarterly NPS prompt condition is met
**Then** they are shown an NPS survey
**And** scores 9-10 provide an opt-in deep link to a G2/Capterra review form.