## Epic 15: Per-Bid Pricing SKU & Pro+ Tier

Introduce a Per-Bid SKU for opportunity-specific upgrades and launch the new Pro+ subscription tier to unlock CRM integrations and dedicated support.
**FRs covered:** FR2.4, FR2.5, FR2.6

### Story 15.1: Stripe One-Time Payment for Per-Bid SKU

As a Bid Manager,
I want to purchase a Per-Bid SKU for a specific opportunity,
So that I can use premium AI features for this bid without upgrading my entire subscription.

**Acceptance Criteria:**

**Given** a user is on an opportunity details page
**When** they initiate purchase of a Per-Bid SKU and complete payment via Stripe
**Then** the opportunity is flagged with `per_bid_sku_active=true`
**And** the purchase is recorded in the billing history

### Story 15.2: Per-Bid Usage Metering Bypass

As a Bid Manager with an active Per-Bid SKU,
I want to generate AI summaries and drafts without hitting tier limits,
So that I am not blocked during a critical pursuit.

**Acceptance Criteria:**

**Given** an opportunity has `per_bid_sku_active=true`
**When** the user invokes a metered AI feature for that opportunity
**Then** the usage metering check bypasses the monthly subscription tier limit
**And** the action proceeds successfully

### Story 15.3: Pro+ Tier Feature Flagging

As a Tenant Admin,
I want to upgrade my subscription to Pro+,
So that I can access CRM integrations and multi-client workspace features.

**Acceptance Criteria:**

**Given** a tenant is on the Pro+ tier
**When** the user accesses the integrations settings
**Then** CRM integration options (HubSpot, Salesforce) are unlocked
**And** the user can configure them
