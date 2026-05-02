# Epic 2: Opportunity Intelligence & Discovery

Users can ingest public tender data, receive smart matches based on their profiles, and get configurable email digests.
**FRs covered:** FR1, FR2, FR3

## Story 2.1: Data Ingestion Pipeline

As a platform admin,
I want the system to ingest public tender data (AOP, TED),
So that users have opportunities to search.

**Acceptance Criteria:**

**Given** active data sources
**When** the Celery crawler job runs
**Then** it ingests tender data into the pipeline schema
**And** uses SELECT FOR UPDATE SKIP LOCKED to prevent race conditions

## Story 2.2: Opportunity Smart Matching

As a Professional user,
I want AI to score opportunities against my profile,
So that I see the most relevant tenders first.

**Acceptance Criteria:**

**Given** my user profile is complete
**When** I view the dashboard
**Then** opportunities are sorted by an AI relevance score
**And** displayed in high-density data tables

## Story 2.3: Email Digest Configuration

As a Bid Manager,
I want to configure daily email alerts,
So that I never miss a relevant opportunity.

**Acceptance Criteria:**

**Given** I have set criteria (CPV, regions)
**When** the daily digest worker runs
**Then** I receive a formatted email with matching opportunities
**And** I can click through directly to the platform