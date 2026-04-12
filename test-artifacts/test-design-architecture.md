---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-12'
workflowType: 'testarch-test-design'
inputDocuments: ['EU_Solicit_PRD_v1.md', 'EU_Solicit_Solution_Architecture_v4.md', 'EU_Solicit_Requirements_Brief_v4.md', 'EU_Solicit_User_Journeys_and_Workflows_v1.md', 'EU_Solicit_Service_Decomposition_Analysis.md']
---

# Test Design for Architecture: EU Solicit System-Level

**Purpose:** Architectural concerns, testability gaps, and NFR requirements for review by Architecture/Dev teams. Serves as a contract between QA and Engineering on what must be addressed before test development begins.

**Date:** 2026-04-12
**Author:** BMad TEA Agent
**Status:** Architecture Review Pending
**Project:** EU Solicit
**PRD Reference:** EU_Solicit_PRD_v1.md
**ADR Reference:** EU_Solicit_Solution_Architecture_v4.md, EU_Solicit_Service_Decomposition_Analysis.md

---

## Executive Summary

**Scope:** System-level architecture review for the EU Solicit SaaS platform, focusing on multi-tenancy, AI-assisted proposal generation, and event-driven service decomposition.

**Business Context** (from PRD):

- **Revenue/Impact:** SaaS subscription model with tiered pricing, add-ons, and VAT compliance.
- **Problem:** Manual procurement proposal drafting is slow and error-prone; EU Solicit automates this with AI and real-time tender sync.
- **GA Launch:** Phase 5 completion (Analytics & Polish).

**Architecture** (from ADR/Decomposition):

- **Key Decision 1:** Service-oriented decomposition (Auth, Billing, Proposal, AI, Analytics, Tender Sync).
- **Key Decision 2:** Event-driven communication via Kafka for decoupled state changes.
- **Key Decision 3:** AI Solicit Service orchestrating OpenAI/Anthropic for proposal generation.

**Expected Scale**:
- Multi-tenant support for hundreds of organizations/consultants.
- High-volume tender synchronization from CAIS and TED.

**Risk Summary:**

- **Total risks**: 6
- **High-priority (≥6)**: 5 risks requiring immediate mitigation
- **Test effort**: ~75–135 hours (~2–3 weeks for 2 QAs)

---

## Quick Guide

### 🚨 BLOCKERS - Team Must Decide (Can't Proceed Without)

**Pre-Implementation Critical Path** - These MUST be completed before QA can write integration tests:

1. **B-01: Stripe Mocking Strategy** - Architecture must provide a deterministic Stripe simulator or clear webhook mocking patterns to test billing without external dependency. (recommended owner: Fintech/Backend)
2. **B-02: Tenant Data Seeding API** - Architecture must provide a way to seed isolated tenant data (organizations, users, roles) via API for parallel E2E testing. (recommended owner: Backend/Auth)
3. **B-03: AI Output Determinism** - Need a "test mode" or prompt snapshotting mechanism for AI Solicit Service to allow for deterministic assertions in CI. (recommended owner: AI Lead)

**What we need from team:** Complete these 3 items pre-implementation or test development is blocked.

---

### ⚠️ HIGH PRIORITY - Team Should Validate (We Provide Recommendation, You Approve)

1. **R-01: Multi-tenancy Isolation** - Recommendation: Implement Row Level Security (RLS) at the DB level and mandate `tenant_id` in all API scopes. QA will build cross-tenant leakage tests. (Approve: Security Lead)
2. **R-05: Tender Sync Reliability** - Recommendation: Implement idempotent consumers for Kafka and a "health-check" API for sync status. QA will use `recurse` for async validation. (Approve: Data Eng)
3. **R-06: Prompt Injection Hardening** - Recommendation: Use LLM-based guardrails and strict input sanitization. QA will run adversarial prompt suites. (Approve: AI Lead)

**What we need from team:** Review recommendations and approve (or suggest changes).

---

### 📋 INFO ONLY - Solutions Provided (Review, No Decisions Needed)

1. **Test strategy**: Multi-level (E2E for core journeys, API for contracts/logic, Unit for calculations).
2. **Tooling**: Playwright (UI/API), Playwright Utils (Auth, Recurse), k6 (Performance).
3. **Tiered CI/CD**: PR (P0/P1), Nightly (P2/P3/Perf), Weekly (Chaos/Security).
4. **Coverage**: ~15+ core scenarios prioritized P0-P3.
5. **Quality gates**: P0 = 100% pass, P1 ≥ 95%.

**What we need from team:** Just review and acknowledge.

---

## For Architects and Devs - Open Topics 👷

### Risk Assessment

**Total risks identified**: 6 (5 high-priority score ≥6, 1 medium, 0 low)

#### High-Priority Risks (Score ≥6) - IMMEDIATE ATTENTION

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **R-01** | **SEC** | Data Leakage (Multi-tenancy) | 2 | 3 | **6** | Implement RLS + Automated E2E isolation tests. | Security Lead | Phase 4 |
| **R-02** | **BUS** | Billing Inaccuracy (Stripe/VAT) | 2 | 3 | **6** | Stripe Mock/Simulator + VAT validation suite. | Fintech PM | Phase 4 |
| **R-03** | **TECH** | AI Hallucinations | 3 | 2 | **6** | Human-in-the-loop review + LLM guardrails. | AI Lead | Phase 5 |
| **R-05** | **DATA** | Tender Sync Failure | 2 | 3 | **6** | Retry logic (Recurse) + Sync health checks. | Data Eng | Phase 4 |
| **R-06** | **SEC** | LLM Prompt Injection | 2 | 3 | **6** | Input sanitization + Prompt hardening. | AI Lead | Phase 4 |

#### Medium-Priority Risks (Score 3-5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| R-04 | PERF | Search Latency (Elasticsearch) | 2 | 2 | 4 | Load testing + Query optimization. | Backend Lead |

---

### Testability Concerns and Architectural Gaps

**🚨 ACTIONABLE CONCERNS - Architecture Team Must Address**

#### 1. Blockers to Fast Feedback (WHAT WE NEED FROM ARCHITECTURE)

| Concern | Impact | What Architecture Must Provide | Owner | Timeline |
| :--- | :--- | :--- | :--- | :--- |
| **External Stripe Dependency** | Flaky/Slow billing tests | Mocking/Simulator interface | Fintech | Pre-Impl |
| **Non-deterministic AI** | Brittle proposal tests | Semantic evaluation hooks / Test prompts | AI Lead | Phase 4 |
| **Event-driven lag** | Brittle async tests | Event trace IDs / Sync status API | Backend | Pre-Impl |

#### 2. Architectural Improvements Needed (WHAT SHOULD BE CHANGED)

1. **PDF Parsability**
   - **Current problem**: Proposals are exported to PDF; validating content is difficult.
   - **Required change**: Expose proposal data via API in JSON format for easier validation alongside PDF generation.
   - **Impact if not fixed**: Testing relies on brittle PDF parsing or manual check.
   - **Owner**: Proposal Service Lead

---

### Testability Assessment Summary

**📊 CURRENT STATE - FYI**

#### What Works Well

- ✅ **Service-Oriented Design**: Allows for isolated testing of Auth, AI, and Billing services.
- ✅ **Observability Stack**: Planned Loki/Jaeger integration will greatly assist in debugging test failures.
- ✅ **API-First Approach**: FastAPI provides automatic Swagger/OpenAPI docs, simplifying contract testing.

#### Accepted Trade-offs (No Action Required)

- **AI Subjectivity**: Validating "persuasiveness" of AI proposals remains partially manual (UAT).
- **Elasticsearch Latency**: Some eventual consistency is accepted for search indexing; tests will use `recurse`.

---

### Risk Mitigation Plans (High-Priority Risks ≥6)

#### R-01: Data Leakage (Multi-tenancy) (Score: 6) - CRITICAL

**Mitigation Strategy:**
1. Enforce Row Level Security (RLS) in PostgreSQL based on `organization_id`.
2. Middleware validation of `tenant_id` on all incoming requests.
3. Automated "Tenant Cross-Talk" suite: Attempt to access Tenant B's UUID using Tenant A's token.

**Owner:** Security Lead
**Timeline:** Phase 4
**Status:** Planned
**Verification:** Automated E2E isolation suite must pass with 100% success.

---

### Assumptions and Dependencies

#### Assumptions
1. Stripe Test Mode is available and stable for the Billing service.
2. AI LLM providers (OpenAI/Anthropic) remain accessible with predictable latency for CI.
3. Kafka infrastructure is provisioned in the staging environment.

#### Dependencies
1. **Auth API Readiness**: Required by Phase 4 for tenant seeding.
2. **CAIS/TED API Access**: Required by Phase 4 for Tender Sync testing.

---

**End of Architecture Document**
