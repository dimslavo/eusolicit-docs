# Epic 2: Intelligence & Discovery

**Goal:** Users can discover matching public tenders through automated ingestion, AI scoring, and email digests.
**FRs covered:** FR1, FR2, FR3

## Stories

### Story 2.1: Automated Data Ingestion Pipeline
As a bid manager,
I want the platform to automatically ingest data from AOP and TED,
So that I can search a centralized database of opportunities.

**Acceptance Criteria:**
**Given** the background Celery workers are running
**When** new data is published on AOP or TED
**Then** the pipeline must ingest it into the PostgreSQL 16 `pipeline` schema using `SELECT FOR UPDATE SKIP LOCKED`
**And** circuit breakers must protect outbound calls to these portals

### Story 2.2: AI Relevance Scoring Engine
As a bid manager,
I want opportunities scored against my company capabilities,
So that I can quickly identify the most relevant tenders to pursue.

**Acceptance Criteria:**
**Given** an ingested tender
**When** the scoring engine evaluates it
**Then** it must calculate a relevance score based on my company profile, certifications, and financials
**And** store this mapping in the `ai` schema

### Story 2.3: Configurable Email Digests
As a bid manager,
I want to receive daily email digests of matched opportunities,
So that I don't have to manually log in to check for new tenders.

**Acceptance Criteria:**
**Given** I have configured my digest preferences (CPV, regions, budget)
**When** new matching tenders are ingested
**Then** the Notification service must generate and send a formatted email digest
**And** update the dispatch logs in the `notification` schema

### Story 2.4: Dashboard Metrics & Discovery
As a user,
I want a dashboard displaying key metrics and recent activity,
So that I have a high-level view of my opportunity pipeline.

**Acceptance Criteria:**
**Given** I log into the platform
**When** the dashboard loads
**Then** I must see metrics cards, a recent activity feed, and an upcoming deadlines calendar snippet
**And** the data fetching must be wrapped in `<QueryGuard>`