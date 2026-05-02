# Epic 1: User Registration & Subscription

Users can create accounts, select a pricing tier, and manage their subscription.
**FRs covered:** FR11, FR13

## Story 1.1: User Registration

As a new user,
I want to register for an account using my email and password,
So that I can access the platform.

**Acceptance Criteria:**

**Given** a valid email and password (max 128 chars)
**When** I submit the registration form
**Then** my account is created
**And** password is hashed asynchronously and I am logged in

## Story 1.2: Free Tier Selection

As a registered user,
I want to select the Free tier,
So that I can evaluate the platform.

**Acceptance Criteria:**

**Given** I am a registered user
**When** I select the Free tier
**Then** my account is provisioned with Free tier access
**And** VIES VAT validation fails-open to pending on timeout

## Story 1.3: Subscription Upgrade

As a free user,
I want to upgrade to Professional tier via Stripe,
So that I can use advanced features.

**Acceptance Criteria:**

**Given** I am on the Free tier
**When** I choose to upgrade
**Then** a Stripe checkout session is created
**And** upon payment success webhook, my access is instantly updated

## Story 1.4: Role-Based Access Setup

As a company admin,
I want to assign roles to my team members,
So that access is appropriately gated.

**Acceptance Criteria:**

**Given** I am a company admin
**When** I invite a user and assign a role
**Then** the user receives an invite
**And** their access is restricted by the TierGate injection