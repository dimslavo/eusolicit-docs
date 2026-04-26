---
stepsCompleted: ["step-01-init", "step-02-discovery", "step-03-success", "step-04-journeys", "step-05-domain", "step-06-innovation", "step-07-project-type", "step-08-scoping", "step-09-functional", "step-10-nonfunctional", "step-11-polish", "step-12-complete"]
inputDocuments: ["EU_Solicit_Requirements_Brief_v5.md", "project-context.md"]
workflowType: 'prd'
---

# Product Requirements Document - EU Solicit

**Author:** Deb (Agentic Workflow)
**Date:** 2026-04-26
**Status:** Approved
**Version:** 1.0

## 1. Executive Summary
**EU Solicit** is a SaaS platform designed to automate the lifecycle of public procurement and EU grant applications, initially targeting Bulgaria and EU-wide opportunities. The platform leverages KraftData's Agentic AI platform for multi-step workflows, parsing, scoring, and proposal generation, aiming to significantly reduce the friction and cost associated with responding to public tenders and grants. MVP clients self-serve and manage bids using AI-generated artifacts.

## 2. Target Audience & Personas
- **Construction / IT / Engineering Firms:** High volume of complex public tenders requiring technical proposal generation and compliance checks.
- **Consulting Firms:** Manage bids across multiple clients; require white-label functionality and high proposal velocity.
- **Municipalities:** Managing EU structural fund applications with limited in-house capacity.

## 3. Product Vision & Scope
The platform provides intelligent tracking, document parsing, AI proposal generation, and compliance checks based on administrative frameworks (e.g., ZOP, EU Directives). 
**Out of Scope for MVP:** Electronic bid submission, direct e-procurement integration (CAIS, TED eSender), and digital signatures.

## 4. Functional Requirements

### 4.1 Opportunity Intelligence & Discovery
- **Data Crawling:** Ingest AOP, TED, and EU grant portals data into a centralized PostgreSQL 16 data store.
- **Smart Matching:** AI relevance scoring against user capabilities, certifications, and financial capacities.
- **Alerts:** Configurable email digests based on CPV, regions, budget, and deadlines.

### 4.2 Document Analysis & Processing
- **AI Parser:** Ingest full tender packages (PDF, DOCX) up to 100MB per file, max 500MB per package.
- **Summarization:** Generate one-page executive summaries of complex RFPs.
- **Checklist Builder:** Extract mandatory requirements into a compliance checklist.
- **Risk Analysis:** Flag unusual or high-risk contractual clauses.

### 4.3 AI Proposal Generation
- **Drafting:** Streamed generation (`run-stream`) of full technical proposals using KraftData vector stores (RAG) and user profiles.
- **Score Simulation:** Predict evaluator scoring and suggest improvements.
- **Compliance Validation:** Automated checks against admin-assigned regulatory frameworks (e.g., ZOP).

### 4.4 Workflow & Account Management
- **Subscription Tiers:** Free, Starter, Professional, Enterprise tiers gated by Stripe. 
- **Calendar Sync:** Google and Outlook sync (Pro/Enterprise) plus an iCal feed.
- **Role-Based Access Control (RBAC):** Company role ceiling (admin > bid_manager > contributor > reviewer > read_only) and entity-level overrides.
- **Audit Logging:** Immutable audit logs capturing user actions, IP, before/after values for POST/PATCH/DELETE endpoints.

## 5. Non-Functional Requirements (NFRs)

- **Availability:** 99.5% uptime SLA.
- **Latency:** All AI generation operations must use Server-Sent Events (SSE) with `run-stream`, no blocking waits. SSE endpoints must implement idle timeout (120s), total timeout (600s), and heartbeat (15s).
- **Security & Cryptography:** 
  - Passwords bounded at `max_length=128` characters. 
  - `hmac.compare_digest()` required for all inbound webhook validations. 
  - AES-256 for data at rest; TLS 1.3 for transit. 
  - `is_active` checked for all token endpoints. 
  - Outbound calls must use circuit breakers wrapping exponential backoff.
- **Concurrency & Reliability:** 
  - Idempotent Redis Streams consumption via DB constraints + `SETNX`.
  - Atomic Lua scripts for rate/usage limits.
  - Celery background tasks use `SELECT FOR UPDATE SKIP LOCKED` for DB queues to prevent race conditions.
- **Data Residency:** All data stored within the EU (GDPR compliance).

## 6. User Stories & Acceptance Criteria

### Epic 1: Onboarding & Subscription (Billing Service)
- **Story 1.1: User Registration & Tier Selection**
  - *As a new user, I want to register and select a subscription tier so that I can access matching opportunities.*
  - **Acceptance Criteria:**
    - User provides email, password (max 128 chars), and company details.
    - Password hashed asynchronously using `run_in_executor`.
    - User is provisioned in Stripe via background task; API returns 201 immediately.
    - VIES VAT validation fails-open to `pending` status on timeout.

- **Story 1.2: Upgrade to Professional Tier**
  - *As a free user, I want to upgrade to Professional so that I can generate AI proposals.*
  - **Acceptance Criteria:**
    - Upgrading triggers a Stripe checkout session.
    - Webhook processor uses `webhook_events` table for idempotency to prevent double billing.
    - Access rights instantly apply post-payment via cache deletion (no `SET` race condition).

### Epic 2: Intelligence & Discovery
- **Story 2.1: Automated Tender Digest**
  - *As a Bid Manager, I want daily email digests of relevant tenders.*
  - **Acceptance Criteria:**
    - Worker queries database for newly ingested AOP/TED opportunities.
    - Matches run against user profile.
    - Redis-backed CELERY signals ensure correlation.
    - User receives correctly formatted email.

### Epic 3: Document Analysis & Summary
- **Story 3.1: Generate Executive Summary**
  - *As a Reviewer, I want an AI-generated executive summary of a 200-page tender.*
  - **Acceptance Criteria:**
    - File upload checks for 100MB limit and ClamAV scanning.
    - Frontend utilizes `<QueryGuard>` around fetching state.
    - KraftData agent triggered via AI Gateway with required `X-Caller-Service` header.
    - Backend updates `shared.audit_log` with the file upload action.

### Epic 4: Proposal Generation (SSE Streaming)
- **Story 4.1: Stream AI Proposal Draft**
  - *As a Contributor, I want the AI to draft my proposal in real-time so I don't have to wait for the full document to finish.*
  - **Acceptance Criteria:**
    - Frontend implements native `fetch` + `ReadableStream` (no Axios/EventSource) to handle SSE.
    - Backend returns `StreamingResponse` with `Cache-Control: no-cache` and `X-Accel-Buffering: no`.
    - Usage limits (UsageGate Lua script) are decremented *before* the generator yields HTTP 200.
    - Connection terminates gracefully with terminal events.

### Epic 5: Compliance Validation
- **Story 5.1: Validate Proposal against ZOP**
  - *As a Bid Manager, I want the system to check my drafted proposal against the assigned compliance framework.*
  - **Acceptance Criteria:**
    - System checks the assigned regulatory framework (e.g., ZOP).
    - Compliance checker agent analyzes the document.
    - Frontend displays pass/fail criteria and improvement suggestions.

### Epic 6: Workflows & Approvals
- **Story 6.1: Bid/No-Bid Decision Matrix**
  - *As an Admin, I want an AI recommendation on bid viability and the ability to override it.*
  - **Acceptance Criteria:**
    - AI provides a recommendation payload (win probability, risks).
    - Admin can submit a manual decision overriding the recommendation.
    - Upsert endpoints partition `decision` fields from `ai_recommendation` fields so one does not clobber the other.
    - Resulting decision is written to audit logs asynchronously using `asyncio.create_task`.

## 7. Open Issues & Future Scope
- Transition from per-instance circuit breaker to Redis-backed circuit breaker for multi-replica scaling.
- Integrate direct e-procurement portal submissions (CAIS, TED eSender).
- Develop digital signature integration.
- Finalize CRM/ERP connector strategies.