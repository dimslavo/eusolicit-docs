# Epic 4: Opportunity Data Pipeline

The system robustly ingests, scans, and parses procurement data daily from AOP and TED into a standardized data model.

### Story 4.1: Daily AOP / TED Crawl
As an Operator,
I need new tenders ingested daily,
So that the platform has up-to-date opportunities.

**Acceptance Criteria:**
**Given** the Celery Beat schedule
**When** the crawler runs
**Then** it pulls data from AOP and TED with a 24-hour freshness SLA
**And** runs are tracked idempotently, using SELECT FOR UPDATE SKIP LOCKED to prevent duplicate processing

### Story 4.2: Document Download & Virus Scan
As a Bid Manager,
I expect every tender document to be safe and accessible,
So that I can review them without security risks.

**Acceptance Criteria:**
**Given** an ingested opportunity
**When** its documents are downloaded
**Then** files up to 100 MB are saved to MinIO and scanned by ClamAV
**And** rejected files surface an error and stuck documents are cleaned up via a Celery Beat task

### Story 4.3: Opportunity Normalisation & Emit
As downstream services,
We need clean structured opportunities on a stream,
So that we can process them for AI scoring and notifications.

**Acceptance Criteria:**
**Given** crawled opportunity data
**When** it is parsed
**Then** CPV codes, deadlines, and budgets are extracted and normalized
**And** an opportunity.created Redis Stream event is emitted idempotently
