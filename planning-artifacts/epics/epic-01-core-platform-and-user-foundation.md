---
epicNumber: 1
title: Core Platform & User Foundation
status: draft
dependencies: []
frsCovered: [FR-1, FR-2, FR-3, FR-4, FR-5, FR-6, FR-7, FR-10, FR-11, FR-12, FR-13, FR-14]
nfrsRelevant: [NFR-5, NFR-6, NFR-7, NFR-16, NFR-18, NFR-19, NFR-20, NFR-21]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
---

## Epic 1: Core Platform & User Foundation

This epic establishes the foundational SaaS capabilities, allowing users to register, manage their accounts and company profiles, and subscribe to paid plans. It is the bedrock upon which all other features are built. After this epic, a new user can sign up (email/password or Google), verify their email, set up a company workspace, invite teammates with role-based permissions, and self-serve a Stripe subscription with a 14-day Professional trial. Feature gating by tier is enforced platform-wide.

### Story 1.1: User Registration with Email and Password

As a new user,
I want to register for an account using my email and password,
So that I can access the platform's features.

**Acceptance Criteria:**

**Given** I am on the registration page
**When** I enter a valid, unique email address and a password that meets the strength requirements (minimum 12 characters, mixed case, number, symbol)
**Then** a new user account is created in the `client.users` table with a hashed password (argon2id)
**And** I am sent a verification email containing a single-use, time-limited token
**And** I am redirected to a "check your email" confirmation page.

**Given** I attempt to register with an email that already exists
**When** I submit the form
**Then** the system returns a generic success response (to prevent user enumeration)
**And** no duplicate account is created.

### Story 1.2: User Login with Email and Password

As a registered user,
I want to log in to my account using my email and password,
So that I can access my workspace and data.

**Acceptance Criteria:**

**Given** I am a registered user with a verified email
**When** I enter my correct email and password on the login page
**Then** I am authenticated and redirected to my dashboard
**And** a JWT access token (15 min TTL) and refresh token (7 day TTL, HTTP-only secure cookie) are issued
**And** the access token contains my `user_id`, active `company_id`, and roles claim.

**Given** I provide invalid credentials
**When** I submit the login form
**Then** I see a generic "invalid email or password" error
**And** the failed attempt is rate-limited (5 attempts per 15 min per IP).

### Story 1.3: User Email Verification

As a new user who has just registered,
I want to verify my email address by clicking a link in an email,
So that I can secure my account and unlock core platform features.

**Acceptance Criteria:**

**Given** I have registered an account but not yet verified my email
**When** I click the unique verification link within its 24-hour validity window
**Then** my email is marked as verified in the database
**And** I am redirected to my dashboard with my account fully activated.

**Given** my verification link has expired
**When** I click the link
**Then** I see an "expired" message with a button to request a new verification email.

**Given** I am unverified
**When** I attempt to access core features (proposals, AI analysis, etc.)
**Then** I am redirected to a "please verify your email" interstitial page.

### Story 1.4: User Registration and Login with Google

As a new or existing user,
I want to register or log in using my Google account,
So that I can access the platform without creating a separate password.

**Acceptance Criteria:**

**Given** I am on the registration or login page
**When** I click the "Sign in with Google" button and complete the Google OAuth 2.0 flow
**Then** the OAuth callback creates a new account if no user exists for that Google email, or logs me into the existing account
**And** my email is automatically marked verified
**And** I am redirected to my dashboard with valid JWT tokens issued.

**Given** my Google email collides with an existing email-password account
**When** the OAuth callback completes
**Then** the account is linked (Google identity stored alongside the email-password identity)
**And** I receive a notification email confirming the link.

### Story 1.5: Company Profile Management

As a company admin,
I want to create and update my company's profile with legal and business information,
So that this data can be used to auto-fill documents and personalize AI features.

**Acceptance Criteria:**

**Given** I am logged in as an admin for my company
**When** I navigate to the company profile page
**Then** I can fill in fields including legal name, VAT number, registration number, address, country, sectors of expertise (CPV codes), key personnel, and certifications
**And** the saved information is stored against my `company_id` in the `client.companies` table
**And** all fields are validated (VAT format per country, CPV against the EU code list).

**Given** I am a non-admin user
**When** I navigate to the company profile page
**Then** I see the profile in read-only mode.

### Story 1.6: User Invitation and Role Management

As a company admin,
I want to invite team members to my company workspace and assign them roles,
So that my team can collaborate on the platform with appropriate permissions.

**Acceptance Criteria:**

**Given** I am an admin in my company's workspace
**When** I enter a team member's email and select a role from {Admin, Bid Manager, Contributor, Reviewer, Read-Only}
**Then** an invitation email is sent containing a unique acceptance link valid for 7 days
**And** a `client.invitations` record is created with status `pending`.

**Given** an invitee accepts a valid invitation
**When** they complete registration (or sign in with an existing account)
**Then** they are added to the workspace with the assigned role
**And** the invitation status changes to `accepted`.

**Given** I am an admin
**When** I change a team member's role or revoke their access
**Then** their permissions update immediately on their next request (cached JWT roles claim is invalidated within 60s).

### Story 1.7: Subscription with Stripe (Starter and Professional)

As a user,
I want to subscribe to a paid plan (Starter or Professional) using my credit card,
So that I can access premium features.

**Acceptance Criteria:**

**Given** I am on the billing or upgrade page
**When** I select a plan and complete Stripe Checkout with valid card details
**Then** a Stripe Subscription is created against my `company_id`
**And** my workspace is upgraded to the selected tier upon receipt of the `customer.subscription.created` webhook
**And** I receive a confirmation email with my invoice.

**Given** Stripe sends a webhook
**When** the system processes it
**Then** the handler is idempotent (duplicate webhook events do not double-charge or duplicate records)
**And** EU VAT is correctly calculated and applied via Stripe Tax.

### Story 1.8: 14-day Free Trial for Professional Tier

As a new user,
I want to be automatically enrolled in a 14-day free trial of the Professional tier upon registration,
So that I can experience the full value of the platform before committing to a paid plan.

**Acceptance Criteria:**

**Given** I have just registered a new account
**When** I log in for the first time
**Then** my company workspace has access to all Professional tier features
**And** the `subscriptions.tier` is set to `professional_trial` with `trial_ends_at` 14 days from registration.

**Given** I am on a trial
**When** I view any page in the application
**Then** a non-dismissible banner shows "X days left in your Professional trial" with an "Upgrade now" CTA.

**Given** my trial expires without payment
**When** the daily trial-expiry job runs
**Then** my workspace is downgraded to the Free tier
**And** I receive an email summarizing what features I lost and how to upgrade.

### Story 1.9: Feature Gating Based on Subscription

As a user on a specific subscription tier,
I want to only see and access the features included in my plan,
So that the user interface is clean and I understand the value of upgrading.

**Acceptance Criteria:**

**Given** I am logged in with a 'Starter' tier subscription
**When** I attempt to access a 'Professional' tier feature
**Then** the API returns `403 FEATURE_GATED` with the required tier
**And** the UI shows a paywall modal with a comparison and an "Upgrade" CTA
**And** the gated UI element is rendered with a lock icon and disabled state.

**Given** I exceed a usage limit (e.g., AI generations per month)
**When** I attempt the metered action
**Then** the system returns `429 USAGE_LIMIT_EXCEEDED` with the limit details
**And** an in-app notification suggests the next tier or an add-on purchase.

### Story 1.10: Self-service Subscription Management

As a paying user,
I want to manage my subscription (upgrade, downgrade, cancel) through a self-service portal,
So that I have control over my billing without contacting support.

**Acceptance Criteria:**

**Given** I am a subscribed user
**When** I click "Manage Subscription" in account settings
**Then** I am redirected to a Stripe Customer Portal session keyed to my `customer_id`
**And** I can view invoices, update payment methods, change plans, and cancel my subscription.

**Given** I cancel my subscription via the portal
**When** Stripe emits the `customer.subscription.updated` webhook
**Then** my workspace remains on the paid tier until `current_period_end`
**And** at period end, my tier is downgraded to Free automatically.

### Story 1.11: One-Time Add-on Purchases

As a user,
I want to purchase one-time add-ons (e.g., extra AI credits, premium templates) without changing my recurring plan,
So that I can flexibly extend my platform usage for specific bids.

**Acceptance Criteria:**

**Given** I am a logged-in user on any paid tier
**When** I click "Buy add-on" for a listed product
**Then** Stripe Checkout opens in one-time payment mode
**And** on successful payment, the add-on entitlement (e.g., +50 AI credits) is granted to my workspace within 30 seconds via webhook processing
**And** the purchase is recorded with full audit trail and is idempotent against duplicate webhooks.
