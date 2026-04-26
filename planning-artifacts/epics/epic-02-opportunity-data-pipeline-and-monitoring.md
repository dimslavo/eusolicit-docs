# Epic 2: Opportunity Data Pipeline & Monitoring

Automatically crawl EU procurement portals, normalize data, and match opportunities to company profiles.

### Story 2.1: Automated Crawling of EU Procurement Portals

As a Bid Manager,
I want the system to automatically crawl AOP, TED, and the EU Funding & Tenders Portal,
So that I don't have to manually monitor thousands of sources.

**Acceptance Criteria:**

**Given** the background Celery task scheduler is active
**When** the scheduled interval triggers
**Then** the system successfully fetches new opportunity listings from AOP, TED, and EU Funding & Tenders
**And** gracefully handles rate limits via exponential backoff.

### Story 2.2: Opportunity Data Normalization

As a Bid Manager,
I want the extracted procurement data normalized into a unified schema,
So that I can search and filter consistently across diverse EU sources.

**Acceptance Criteria:**

**Given** raw data fetched from procurement portals
**When** the KraftData Agent Team processes the data
**Then** it normalizes the CPV codes, budget thresholds, and deadlines
**And** stores the structured data in the unified PostgreSQL database schema.

### Story 2.3: Opportunity Matching and Scoring

As a Bid Manager,
I want the system to score opportunities against my company profile and past bids,
So that I am alerted only to highly relevant tenders.

**Acceptance Criteria:**

**Given** normalized opportunity listings in the system
**When** the scoring algorithm runs for my active workspace
**Then** the system assigns a relevance score based on past bids and profile fit
**And** tags opportunities as `platform-attributed` for analytics.