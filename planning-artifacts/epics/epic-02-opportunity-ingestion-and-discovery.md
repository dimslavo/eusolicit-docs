---
epicNumber: 2
title: Opportunity Ingestion & Discovery
status: draft
dependencies: [1]
frsCovered: [FR-15, FR-16, FR-17, FR-18, FR-19, FR-20, FR-40]
nfrsRelevant: [NFR-1, NFR-3, NFR-7, NFR-12, NFR-13, NFR-23]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
---

## Epic 2: Opportunity Ingestion & Discovery

This epic delivers the first core value proposition: helping users find relevant procurement opportunities. It covers the backend data pipeline (Bulgarian AOP and EU TED crawlers, KraftData enrichment, AI relevance scoring) and the user-facing list/detail/save/digest experience. After this epic, an authenticated user with a populated company profile can discover, filter, sort, and save opportunities, and subscribe to email digests for new matches.

### Story 2.1: Data Pipeline for AOP and TED Ingestion

As a platform operator,
I want the system to automatically ingest new procurement opportunities from AOP (Bulgaria) and TED (EU),
So that users have access to a comprehensive and up-to-date database of tenders.

**Acceptance Criteria:**

**Given** the data-pipeline service is deployed
**When** the scheduled crawler runs (every 6 hours for AOP, daily for TED)
**Then** new and updated opportunities are parsed, normalized into the `pipeline.opportunities` schema, and emitted on the `opportunities` Redis stream as `OpportunitiesIngested` events
**And** the crawler handles transient HTTP failures with exponential backoff retry (3 attempts) and circuit breaker
**And** ingestion metrics (records processed, errors, duration) are exported to Prometheus.

**Given** a source portal returns malformed data for one record
**When** the crawler processes the batch
**Then** the offending record is logged with full context and skipped
**And** the rest of the batch completes successfully
**And** the error rate metric increments to alert operators.

### Story 2.2: Opportunity Search and Filtering

As a user,
I want to search opportunities using keywords and apply filters like country, budget range, deadline, and CPV code,
So that I can quickly find tenders relevant to my business.

**Acceptance Criteria:**

**Given** I am on the Opportunities page
**When** I enter a search term in the search box
**Then** results update with debounced full-text search across title, description, and contracting authority within 300ms p95
**And** matched terms are highlighted in the list.

**Given** I select filter facets (country, CPV, budget min/max, deadline range, status)
**When** any facet changes
**Then** the result list, count, and facet aggregations update accordingly
**And** active filters are shown as removable chips above the list
**And** the URL reflects the filter state for shareable links.

### Story 2.3: AI Relevance Scoring

As a user,
I want to see an AI-generated relevance score for each opportunity,
So that I can prioritize tenders based on my company's profile.

**Acceptance Criteria:**

**Given** my company profile is complete with sectors of expertise (CPV) and capabilities
**When** a new opportunity is ingested or my profile is updated
**Then** the AI gateway computes a relevance score (0--100) per opportunity per company
**And** the score is stored in `client.opportunity_scores` keyed by `(company_id, opportunity_id)`
**And** stale scores are recomputed within 1 hour of any profile or opportunity change.

**Given** I view the opportunity list
**When** scores are available
**Then** each opportunity displays its score with a color-coded indicator (red <40, amber 40--70, green >70)
**And** I can sort the list by score descending.

### Story 2.4: Opportunity List and Detail View

As a user,
I want to view opportunities in a clear list and click into one to see full details,
So that I can efficiently browse and evaluate potential tenders.

**Acceptance Criteria:**

**Given** I am on the Opportunities page
**When** the page loads
**Then** I see a paginated list (50 per page) showing title, contracting authority, country, CPV, budget, deadline, status, and relevance score
**And** the list supports sort by relevance, deadline, budget, or recently published
**And** infinite-scroll or paginated controls work for large result sets.

**Given** I click an opportunity row
**When** the detail page renders
**Then** I see all metadata, the full description, attached documents, evaluation criteria, and links to source portal
**And** Free-tier users see a paywall on locked fields (budget, full documents, AI analysis) with an upgrade CTA.

### Story 2.5: Save and Manage Starred Opportunities

As a user,
I want to "star" opportunities to save them for future reference,
So that I can build a shortlist to work on.

**Acceptance Criteria:**

**Given** I am viewing the opportunity list or detail
**When** I click the star icon
**Then** the opportunity is added to my saved list (`client.saved_opportunities` for `(user_id, opportunity_id)`)
**And** the star icon toggles to filled state with optimistic UI update
**And** I can filter the main list to show only my starred items.

**Given** I unstar an opportunity
**When** I click the filled star
**Then** the row is removed from my saved list with a 5-second undo toast
**And** the change is persisted on toast dismissal.

### Story 2.6: Email Digests for New Opportunities

As a user,
I want to configure and receive email digests of new opportunities matching my saved searches,
So that I can stay informed without logging in daily.

**Acceptance Criteria:**

**Given** I have created a saved search with filters
**When** I enable email digests for it and choose a frequency (daily or weekly)
**Then** a `client.digest_subscriptions` record is created
**And** the daily/weekly scheduled job evaluates each subscription, queries new matching opportunities since last send, and emits a `NotificationRequested` event with channel `email` and template `opportunity_digest`.

**Given** there are zero new matches in a period
**When** the digest job runs
**Then** no email is sent
**And** the subscription's `last_evaluated_at` is updated.

**Given** I receive a digest
**When** I click "Unsubscribe" in the email footer
**Then** the digest subscription is disabled immediately and confirmed on a landing page.
