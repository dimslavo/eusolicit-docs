# E08: Subscription & Billing

**Sprint**: 9--10 | **Points**: 55 | **Dependencies**: E02, E06 | **Milestone**: Beta

## Goal

Implement the full subscription and billing lifecycle for EU Solicit using Stripe as the payment backbone. This includes tiered subscription management (Free, Starter, Professional, Enterprise), a 14-day no-credit-card trial of the Professional tier for new registrations, per-bid add-on purchases, EU VAT handling via Stripe Tax, enterprise custom invoicing, and usage metering with Redis-backed atomic counters. Upon completion, companies can self-serve their entire billing lifecycle while the platform enforces tier-based feature gating against real subscription state rather than mocks.

## Acceptance Criteria

- [ ] New company registration creates a Stripe customer and auto-provisions a 14-day Professional trial subscription (no payment method required)
- [ ] Trial eligibility is enforced as one trial per company; duplicate attempts are rejected
- [ ] Trial expiry downgrades the company to the Free tier, preserves all data, and gates premium features
- [ ] A `customer.subscription.trial_will_end` webhook triggers a reminder email event 3 days before trial end
- [ ] Users can upgrade/downgrade via Stripe Checkout redirect and return to a success or cancel page
- [ ] Stripe Customer Portal link is accessible for self-service plan changes, cancellation, and payment method updates
- [ ] All subscription webhook events (created, updated, deleted, invoice.paid, invoice.payment_failed) sync correctly to the local `subscriptions` table
- [ ] Per-bid add-on purchases complete via Stripe Checkout (mode: payment), and the `checkout.session.completed` webhook creates an `add_on_purchases` record that unlocks the feature for the specific opportunity
- [ ] EU VAT is calculated automatically via Stripe Tax on all charges; tax IDs are collected, synced to Stripe, and validated via VIES
- [ ] Enterprise custom invoices can be created via admin tooling using the Stripe Invoicing API with NET 30/60 terms
- [ ] Usage metering (AI summaries, proposal drafts, compliance checks) uses Redis atomic counters per company per billing period, with a daily Celery task syncing to Stripe usage records
- [ ] A `subscription.changed` event is published to Redis Streams on every tier change, triggering cache invalidation across services
- [ ] Pricing page displays an accurate tier comparison table with feature matrix
- [ ] Trial banner in the topbar shows days remaining and an upgrade CTA throughout the trial period
- [ ] Subscription management page shows current plan, usage meters (consumed vs. limit), portal link, and billing history

## Stories

### S08.01: Stripe Customer Provisioning on Company Registration
**Points**: 3 | **Type**: backend

Create a Stripe customer automatically when a new company registers. Sync the company name, email, and tax ID (VAT number) to the Stripe customer object. Store the `stripe_customer_id` on the local `subscriptions` record. Handle idempotency so repeated calls do not create duplicate Stripe customers. Wire into the existing E02 registration flow as a post-registration hook.

---

### S08.02: Subscription & Tier Database Schema
**Points**: 3 | **Type**: backend

Create Alembic migrations for the `subscriptions`, `tier_access_policies`, and `add_on_purchases` tables in the client schema. Seed `tier_access_policies` with the four tiers and their feature limits (max regions, max CPV sectors, max budget threshold, AI summaries limit, proposal drafts limit, compliance checks limit, max team members, calendar sync, API access, whitelabel). Add foreign key from `subscriptions.company_id` to companies. Add indexes on `stripe_subscription_id` and `stripe_customer_id`.

---

### S08.03: 14-Day Professional Trial Provisioning
**Points**: 3 | **Type**: backend

On company registration, create a Stripe subscription on the Professional price with `trial_end` set to 14 days out and `payment_behavior: default_incomplete` (no payment method required). Store `status: trialing`, `trial_start`, and `trial_end` in the local `subscriptions` table. Enforce one-trial-per-company by checking for any prior subscription record before provisioning. Publish a `subscription.changed` event to Redis Streams.

---

### S08.04: Stripe Webhook Endpoint & Subscription Lifecycle Sync
**Points**: 5 | **Type**: backend

Implement a Stripe webhook receiver endpoint with signature verification (`stripe.Webhook.construct_event`). Handle `customer.subscription.created`, `customer.subscription.updated`, and `customer.subscription.deleted` events by upserting the local `subscriptions` table (tier, status, period dates). Handle `invoice.paid` to confirm active status and `invoice.payment_failed` to mark `past_due`. Implement idempotent event processing with a `webhook_events` deduplication table. Log all incoming events for audit.

---

### S08.05: Trial Expiry Handling & Downgrade Logic
**Points**: 3 | **Type**: backend

Handle `customer.subscription.trial_will_end` by publishing a `trial.expiring` event for the notification service (reminder email 3 days before). On trial expiry without payment method (subscription transitions to `past_due` or `canceled`), update local subscription to `tier: free` and `status: active`. Preserve all company data but enforce Free-tier feature limits via the existing E06 tier gating middleware. Publish `subscription.changed` to Redis Streams for cache invalidation.

---

### S08.06: Stripe Checkout Upgrade/Downgrade Flow
**Points**: 5 | **Type**: fullstack

Build the Stripe Checkout redirect flow for subscription creation and tier changes. Backend: create a Checkout Session (mode: subscription) with the selected price, pre-filled customer, and success/cancel URLs. Frontend: tier selection triggers an API call, receives the Checkout URL, and redirects. On return, the success page polls for webhook confirmation and displays a confirmation message. Handle proration for mid-cycle upgrades/downgrades via Stripe's default proration behavior.

---

### S08.07: Stripe Customer Portal Integration
**Points**: 2 | **Type**: fullstack

Backend: generate a Stripe Customer Portal session URL scoped to the authenticated company's `stripe_customer_id`. Configure the portal via Stripe Dashboard or API to allow plan changes, cancellation, and payment method updates. Frontend: render a "Manage Subscription" button on the subscription management page that opens the portal in a new tab. On portal return, refresh subscription state from the local database.

---

### S08.08: Usage Metering with Redis Counters & Stripe Sync
**Points**: 5 | **Type**: backend

Implement Redis-based atomic counters (INCR) per company per billing period for AI summaries, proposal drafts, and compliance checks. Key format: `usage:{company_id}:{period_start}:{metric}`. Expose an internal `usage.increment(company_id, metric)` function called by the respective feature services. Create a daily Celery beat task that reads counters and reports to Stripe via `stripe.SubscriptionItem.create_usage_record`. Reset counters on period rollover. Add a `/api/v1/subscription/usage` endpoint returning current consumption vs. tier limits.

---

### S08.09: Per-Bid Add-On Purchase Flow
**Points**: 3 | **Type**: fullstack

Backend: create a Stripe Checkout Session (mode: payment) for add-on types (proposal generation, deep compliance audit, pricing analysis) linked to a specific `opportunity_id`. On `checkout.session.completed` webhook, insert into `add_on_purchases` with the payment intent ID and amount. Frontend: render "Purchase [Add-On]" buttons on the opportunity detail page (visible when the feature is not included in the current tier). On click, redirect to Stripe Checkout. On success return, display confirmation and unlock the feature for that opportunity.

---

### S08.10: EU VAT Handling via Stripe Tax & VIES Validation
**Points**: 5 | **Type**: fullstack

Enable Stripe Tax on all Checkout Sessions and subscriptions for automatic EU VAT calculation based on customer country and business status. Backend: on registration or profile update, accept VAT number input, validate against the EU VIES service (async with retry), and sync to `stripe.Customer.tax_ids`. Store validation status locally. Frontend: add a VAT number input field on the company profile page and during checkout, with a real-time validation status indicator (pending, valid, invalid). Handle B2B reverse-charge scenarios where Stripe Tax applies 0% VAT.

---

### S08.11: Enterprise Custom Invoicing
**Points**: 3 | **Type**: backend

Build an admin-facing API for creating Stripe invoices for Enterprise-tier customers with custom payment terms (NET 30 or NET 60). Use the Stripe Invoicing API to create draft invoices with line items, set `days_until_due`, and finalize/send. Handle `invoice.paid` webhook to update local subscription status. Store invoice metadata linking to the company and admin who created it. Expose a list endpoint for admin to view all Enterprise invoices and their payment status.

---

### S08.12: Pricing Page & Tier Comparison UI
**Points**: 3 | **Type**: frontend

Build a public-facing pricing page with a tier comparison table showing Free, Starter, Professional, and Enterprise plans. Display monthly prices, feature limits (regions, CPV sectors, AI summaries, proposal drafts, compliance checks, team members), and boolean features (calendar sync, API access, whitelabel). Highlight the Professional tier as "Most Popular." Add CTA buttons: "Start Free Trial" (Professional), "Get Started" (Starter), "Contact Sales" (Enterprise). Responsive layout for mobile. Pull tier data from the `tier_access_policies` API endpoint.

---

### S08.13: Subscription Management Page & Trial Banner
**Points**: 5 | **Type**: frontend

Build the subscription management dashboard showing: current tier and status, billing period, usage meters as progress bars (AI summaries, proposal drafts, compliance checks consumed vs. limit), a "Manage Subscription" button linking to the Stripe Customer Portal, and a billing history table from local invoice records. Implement a persistent trial banner in the global topbar that appears during the `trialing` status, showing "X days remaining in your Professional trial" with a countdown and a prominent "Upgrade Now" CTA. Hide the banner after trial conversion or downgrade.

---

### S08.14: Subscription Changed Event & Tier Cache Invalidation
**Points**: 2 | **Type**: backend

On every subscription tier or status change (detected in webhook handlers), publish a `subscription.changed` event to Redis Streams containing `company_id`, `old_tier`, `new_tier`, and `timestamp`. Implement a consumer in the API gateway / tier gating middleware that listens for these events and invalidates the cached tier for the affected company. Ensure the tier gating middleware (E06) falls back to a database lookup on cache miss so that feature access is immediately consistent after a plan change.
