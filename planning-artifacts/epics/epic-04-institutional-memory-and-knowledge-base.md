# Epic 4: Institutional Memory & Knowledge Base

Allow users to upload past credentials and track bid outcomes to improve future AI generations via RAG.

### Story 4.1: Upload and Manage Company Credentials

As a Bid Manager,
I want to upload past credentials, references, and CVs to my workspace,
So that the AI uses accurate institutional memory for generating proposals.

**Acceptance Criteria:**

**Given** I am in my active workspace
**When** I upload credential documents
**Then** they are securely processed into the workspace's vector storage vault
**And** remain completely isolated from other tenant workspaces.

### Story 4.2: Track Bid Outcomes and Execute Lessons Learned

As a Bid Director,
I want to log bid outcomes and trigger the Lessons Learned Agent,
So that the knowledge base is updated with win/loss data to improve future drafts.

**Acceptance Criteria:**

**Given** a completed proposal in the workspace
**When** I log the outcome (won, lost, withdrawn) with optional evaluator scores
**Then** the Lessons Learned Agent processes the feedback
**And** updates the RAG context within the vector store for future queries.