# Epic 5: Opportunity Discovery

Users can efficiently search, filter, and score parsed opportunities to surface relevant bid matches and decide whether to pursue them.

### Story 5.1: Search & Filter
As a user,
I want to find tenders quickly,
So that I can focus on the best opportunities.

**Acceptance Criteria:**
**Given** I am on the search page
**When** I apply filters and text queries
**Then** PostgreSQL FTS performs the query and returns paginated results
**And** filter state is URL-driven and Free-tier constraints are enforced at the field level

### Story 5.2: Match Score & Inbox
As a Bid Manager,
I want a daily-curated inbox,
So that I can see the best-matching opportunities without searching.

**Acceptance Criteria:**
**Given** I have configured my company profile
**When** I view my matches inbox
**Then** opportunities are scored 0-100 by AI relevance
**And** matches above my threshold are sorted by score descending, then deadline ascending

### Story 5.3: Opportunity Detail
As a user,
I want everything about a tender on one page,
So that I can evaluate it fully.

**Acceptance Criteria:**
**Given** I view an opportunity detail page
**When** the page loads
**Then** it displays an AI executive summary, requirements checklist, and risk highlights
**And** Bid / No-Bid actions are available for bid_manager roles and above
