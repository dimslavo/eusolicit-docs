# Epic 2: Opportunity Ingestion & Discovery

This epic delivers the first core value proposition: helping users find relevant procurement opportunities. It includes the backend data pipeline for ingesting tenders and the user-facing interface for searching, filtering, and saving them.

**FRs covered:** FR-15, FR-16, FR-17, FR-18, FR-19, FR-20, FR-40

## Stories

### Story 2.1: Data Pipeline for AOP and TED
As a platform operator,
I want the system to automatically ingest new procurement opportunities from AOP (Bulgaria) and TED (EU),
So that users have access to a comprehensive and up-to-date database of tenders.

**Acceptance Criteria:**
**Given** the data pipeline service is running
**When** the scheduled crawlers execute
**Then** new and updated opportunities from AOP and TED are parsed and stored in the database
**And** the ingestion process is logged and monitored for errors.

### Story 2.2: Opportunity Search and Filtering
As a user,
I want to search for opportunities using keywords and apply filters like country, budget, and CPV code,
So that I can quickly find tenders that are relevant to my business.

**Acceptance Criteria:**
**Given** I am on the "Opportunities" page
**When** I enter a search term or select filter criteria
**Then** the list of opportunities updates in real-time to show only matching results
**And** active filters are clearly displayed and can be easily removed.

### Story 2.3: AI Relevance Scoring
As a user,
I want to see an AI-generated relevance score for each opportunity,
So that I can prioritize which tenders to focus on based on my company's profile.

**Acceptance Criteria:**
**Given** my company profile is complete with my sectors of expertise
**When** I view the list of opportunities
**Then** each opportunity displays a relevance score (e.g., 85%)
**And** the list can be sorted by this relevance score.

### Story 2.4: Opportunity List and Detail View
As a user,
I want to view opportunities in a clear list and click on one to see its full details,
So that I can efficiently browse and evaluate potential tenders.

**Acceptance Criteria:**
**Given** I am on the "Opportunities" page
**When** I view the list, I see key information like title, authority, deadline, and budget
**And** when I click on an opportunity, I am taken to a detail page with all available information, including attached documents.

### Story 2.5: Save/Star Opportunities
As a user,
I want to be able to "star" or save opportunities that I am interested in,
So that I can easily find them later and create a shortlist to work on.

**Acceptance Criteria:**
**Given** I am viewing a list of opportunities or an opportunity's detail page
**When** I click the "star" icon
**Then** the opportunity is added to my saved list
**And** I can filter the main opportunity list to see only my saved items.

### Story 2.6: Email Digests for New Opportunities
As a user,
I want to configure and receive a daily or weekly email digest of new opportunities that match my saved search criteria,
So that I can stay informed about relevant tenders without having to log in every day.

**Acceptance Criteria:**
**Given** I have saved a search with specific filters
**When** I enable email digests for that search
**Then** the system sends me a periodic email containing a list of new opportunities that match my criteria.
