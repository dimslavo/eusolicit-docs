---
epic: 3
title: Tender Discovery and Data Ingestion
status: draft
source: eusolicit-docs/planning-artifacts/epics.md
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
frs_covered: ["FR-15", "FR-16", "FR-17", "FR-18", "FR-19", "FR-20"]
nfrs_relevant: ["NFR-1", "NFR-3", "NFR-4", "NFR-13", "NFR-23"]
ux_drs_relevant: ["UX-DR1", "UX-DR3"]
---

# Epic 3: Tender Discovery and Data Ingestion

**Phase**: MVP | **Dependencies**: Epic 1 (Identity), Epic 2 (Tier gates), `data-pipeline` service, `ai-gateway` service

## Goal

Users can find relevant public procurement opportunities via a continually updated database fed by automated AOP and TED ingestion, then refine results with full-text search, faceted filters, AI-powered relevance scoring, and personal saved lists.

## User Outcome

After this epic, a user with a populated company profile lands on `/opportunities`, sees a deadline-first feed of opportunities ranked by AI relevance score, can filter by CPV/region/budget, drill into an opportunity's detail page, and star opportunities for later.

## Epic Acceptance Criteria

- [ ] AOP and TED ingestion runs on a Celery beat schedule and writes deduplicated records to `pipeline.opportunities`.
- [ ] The opportunity list endpoint returns p95 < 200ms for the first page (NFR-1) for a workspace with up to 100k matching rows.
- [ ] Relevance score is recalculated whenever (a) a new opportunity is ingested, or (b) the company profile changes; the recalculation is event-driven via the shared event bus.
- [ ] Search supports full-text queries against title, description, and contracting authority, plus exact-match faceted filters.
- [ ] The opportunity list page meets a Lighthouse Performance score >= 90 (NFR-3) when virtualised at 100 rows.
- [ ] Star/save state is per-user-per-workspace and persists across sessions.
- [ ] Crawler health, error rate, and last-run timestamp are exported to Prometheus for ops visibility (NFR-23).

## Stories

### Story 3.1: Automated Ingestion from AOP and TED

As the platform data pipeline,
I want to automatically ingest new opportunities from AOP and TED on a recurring schedule,
So that the database always contains the latest public procurement records.

**FR coverage**: FR-15

**Acceptance Criteria:**

**Given** the Celery beat schedule is active in `data-pipeline`
**When** the AOP and TED crawler jobs run
**Then** new records are fetched, normalised to the shared `OpportunityDTO` schema, and upserted into `pipeline.opportunities` keyed by `(source, source_id)`
**And** previously-seen records are not duplicated.
**And** failed fetches are retried with exponential backoff and surfaced to the admin crawler-status dashboard.
**And** an `OpportunitiesIngested` event is published to the shared bus per batch for downstream relevance scoring.

---

### Story 3.2: Full-Text Search and Filtering

As a user,
I want to search and filter opportunities by keywords, CPV, country, region, and budget,
So that I can narrow the list to relevant tenders.

**FR coverage**: FR-16, FR-18

**Acceptance Criteria:**

**Given** I am on the opportunities list
**When** I apply filters or enter search terms in the global Cmd+K command menu or sidebar
**Then** the list updates to show matching results within the p95 latency budget (< 200ms server, < 500ms perceived)
**And** the URL query string updates so the filter state is shareable and back-button friendly.
**And** facet counts are recomputed for each filter set.
**And** sorts are available for "Relevance" (default) and "Deadline (soonest first)".

---

### Story 3.3: AI-Generated Relevance Scoring

As a user,
I want to see an AI-generated relevance score for each opportunity,
So that I can quickly gauge if a tender is a good fit for my company.

**FR coverage**: FR-17

**Acceptance Criteria:**

**Given** my company profile is populated with sectors and CPV codes
**When** new opportunities are ingested or my profile changes
**Then** the AI gateway calculates a relevance score (0–100) per (company, opportunity) pair
**And** scores are persisted in `pipeline.opportunity_scores` with a `computed_at` timestamp.
**And** scores are visible alongside each opportunity row with a colour-coded badge.
**And** scoring failures fall back to a null score with a "—" badge rather than blocking the row.

---

### Story 3.4: Opportunity Detailed View

As a user,
I want to view the full details of a single opportunity,
So that I can review its deadlines, budget, contracting authority, and source documents.

**FR coverage**: FR-19

**Acceptance Criteria:**

**Given** I click an opportunity in the list
**When** the detail page loads at `/opportunities/[id]`
**Then** I see structured fields (title, deadline countdown, budget, CPV codes, contracting authority, source URL)
**And** tabs for "Overview", "AI Analysis" (gated by tier), and "Documents".
**And** the page is server-rendered for SEO/perf and uses `<QueryGuard>` for tab-level fetches.
**And** unauthenticated access returns 401 and redirects to login.

---

### Story 3.5: Saving Opportunities

As a user,
I want to save or "star" opportunities,
So that I can easily track and return to them later.

**FR coverage**: FR-20

**Acceptance Criteria:**

**Given** I am viewing an opportunity (list row or detail page)
**When** I click the star icon
**Then** a `client.saved_opportunities` row is created/removed for `(user_id, workspace_id, opportunity_id)`
**And** the icon state updates optimistically with rollback on server error.
**And** I can filter the list to "Saved only" via a sidebar toggle.

## Implementation Notes

- Crawlers are isolated in `data-pipeline` and never write outside `pipeline.*` schema.
- Full-text search uses Postgres `tsvector` with a generated column on `pipeline.opportunities` plus GIN index — no external search engine for MVP.
- Relevance scoring is invoked async via the AI gateway with circuit breaker + retry; avoid synchronous coupling between ingestion and scoring.
- Score recompute on profile change is rate-limited per company to prevent thrash.
- All list endpoints implement keyset pagination, not offset, for stable scroll under heavy ingestion.
