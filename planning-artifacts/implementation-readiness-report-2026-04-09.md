---
stepsCompleted: [step-01-document-discovery, step-02-prd-analysis, step-03-epic-coverage-validation, step-04-ux-alignment, step-05-epic-quality-review, step-06-final-assessment]
documentsAssessed:
  prd: eusolicit-docs/EU_Solicit_PRD_v1.md
  architecture: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md
  ux_supplement: eusolicit-docs/EU_Solicit_UX_Supplement_v1.md
  user_journeys: eusolicit-docs/EU_Solicit_User_Journeys_and_Workflows_v1.md
  epics:
    - planning-artifacts/epic-01-infrastructure-foundation.md
    - planning-artifacts/epic-02-authentication-identity.md
    - planning-artifacts/epic-03-frontend-shell-design-system.md
    - planning-artifacts/epic-04-ai-gateway-service.md
    - planning-artifacts/epic-05-data-pipeline-ingestion.md
    - planning-artifacts/epic-06-opportunity-discovery.md
    - planning-artifacts/epic-07-proposal-generation.md
    - planning-artifacts/epic-08-subscription-billing.md
    - planning-artifacts/epic-09-notifications-alerts-calendar.md
    - planning-artifacts/epic-10-collaboration-tasks-approvals.md
    - planning-artifacts/epic-11-grants-compliance.md
    - planning-artifacts/epic-12-analytics-admin-platform.md
---

# Implementation Readiness Assessment Report

**Date:** 2026-04-09
**Project:** EU Solicit
**Assessor Role:** Expert Product Manager & Scrum Master — Requirements Traceability Specialist
**Assessment Version:** 4 (supersedes previous reports dated 2026-04-05)

---

## 1. Document Discovery

### 1.1 Documents Inventoried

| Document Type | File | Status |
|---|---|---|
| **PRD** | `eusolicit-docs/EU_Solicit_PRD_v1.md` | ✅ Found — Whole document |
| **Architecture** | `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md` | ✅ Found — Whole document (v4) |
| **UX Design — Supplement** | `eusolicit-docs/EU_Solicit_UX_Supplement_v1.md` | ✅ Found — Whole document |
| **UX Design — User Journeys** | `eusolicit-docs/EU_Solicit_User_Journeys_and_Workflows_v1.md` | ✅ Found — Whole document |
| **Requirements Brief** | `eusolicit-docs/EU_Solicit_Requirements_Brief_v4.md` | ℹ️ Supporting document (PRD source) |
| **Gap Analysis** | `eusolicit-docs/architecture-vs-requirements-gap-analysis.md` | ℹ️ Prior analysis artifact |
| **Epic E01** | `planning-artifacts/epic-01-infrastructure-foundation.md` | ✅ Found |
| **Epic E02** | `planning-artifacts/epic-02-authentication-identity.md` | ✅ Found |
| **Epic E03** | `planning-artifacts/epic-03-frontend-shell-design-system.md` | ✅ Found |
| **Epic E04** | `planning-artifacts/epic-04-ai-gateway-service.md` | ✅ Found |
| **Epic E05** | `planning-artifacts/epic-05-data-pipeline-ingestion.md` | ✅ Found |
| **Epic E06** | `planning-artifacts/epic-06-opportunity-discovery.md` | ✅ Found |
| **Epic E07** | `planning-artifacts/epic-07-proposal-generation.md` | ✅ Found |
| **Epic E08** | `planning-artifacts/epic-08-subscription-billing.md` | ✅ Found |
| **Epic E09** | `planning-artifacts/epic-09-notifications-alerts-calendar.md` | ✅ Found |
| **Epic E10** | `planning-artifacts/epic-10-collaboration-tasks-approvals.md` | ✅ Found |
| **Epic E11** | `planning-artifacts/epic-11-grants-compliance.md` | ✅ Found |
| **Epic E12** | `planning-artifacts/epic-12-analytics-admin-platform.md` | ✅ Found |

### 1.2 Duplicate Document Issues

⚠️ **WARNING — Prior readiness reports in same folder:**
Three previous reports exist (`implementation-readiness-report-2026-04-05.md`, `...v2.md`, `...v3.md`). These are historical artifacts and do not conflict with planning documents. No action required — this report supersedes them.

No duplicate PRD/Architecture/Epic document formats found. Assessment can proceed cleanly.

---

## 2. PRD Analysis

### 2.1 PRD Structure

The PRD (`EU_Solicit_PRD_v1.md`) is a well-structured requirements document covering:
- Product Summary & Business Model
- Target Users & 5 Personas (Free Explorer, Starter User, Professional User, Enterprise User, Platform Admin)
- Feature Areas 3.1–3.10 with acceptance criteria per feature
- Non-Functional Requirements (Section 4)
- Success Metrics (Section 5)
- Release Milestones: Demo (Sprint 8), Beta (Sprint 12), MVP Launch (Sprint 14)
- Out-of-Scope (Section 7)
- Technical Architecture Reference (Section 8)

### 2.2 Functional Requirements Extracted

> **Note:** The PRD does not use numbered FR1/FR2/FR3 identifiers. Requirements are organized as feature tables per section. The following summarizes all functional requirement areas with feature counts:

| Section | Area | Feature Count |
|---|---|---|
| 3.1 | Opportunity Discovery & Intelligence | 11 features |
| 3.2 | Document Analysis & Intelligence | 4 features |
| 3.3 | Proposal Generation & Optimization | 10 features |
| 3.4 | Proposal Collaboration & Workflow | 9 features |
| 3.5 | EU Grant Specialization | 5 features |
| 3.6 | Compliance & Regulatory Intelligence | 6 features |
| 3.7 | Notifications, Alerts & Calendar | 7 features |
| 3.8 | Analytics & Reporting | 7 features |
| 3.9 | Subscription & Billing | 8 features |
| 3.10 | Platform Administration | 7 features |
| **TOTAL** | | **74 features across 10 areas** |

### 2.3 Non-Functional Requirements Extracted

| # | NFR | Target | Measurement |
|---|---|---|---|
| NFR1 | Availability | 99.5% uptime | Grafana uptime monitoring |
| NFR2 | API latency (p95) | < 200ms REST, < 500ms TTFB for SSE | Prometheus histograms |
| NFR3 | Data residency | All data within EU (AWS eu-central-1) | Infrastructure audit |
| NFR4 | Security | JWT RS256, TLS 1.3, AES-256 at rest, ClamAV | Security audit checklist |
| NFR5 | Scalability | 10K+ active tenders, concurrent agent execution per tenant | Load test |
| NFR6 | GDPR compliance | Right to erasure, DPAs with KraftData, data processing records | Compliance checklist |
| NFR7 | Agent quality | KraftData eval-runs for continuous quality monitoring | Eval score dashboards |
| NFR8 | Internationalisation | Bulgarian + English in MVP; AI outputs in user's language | Manual QA |

**Total NFRs: 8**

### 2.4 PRD Completeness Assessment

**Strengths:**
- Feature areas map cleanly to epics
- Each feature has explicit acceptance criteria and tier gating
- Release milestones define a clear delivery progression
- Out-of-scope section avoids scope creep

**Gaps Identified:**
- **No numbered FR identifiers** — the PRD does not assign FR numbers (FR1, FR2…). This makes requirements traceability between PRD features and epic stories entirely implicit and informal, relying on section headings rather than traceable IDs.
- **No explicit GDPR implementation requirements** — NFR6 mentions GDPR but no feature in sections 3.1–3.10 specifies the erasure API, DPA flows, or consent management UI.
- **Enterprise API spec absent** — Section 3.10 mentions "RESTful API documented via OpenAPI" for Enterprise tier but no specification of endpoints, authentication model, or rate limits exists in the PRD.
- **Demo-milestone billing simulation not specified** — Section 6.1 states billing is "simulated (tier set manually in DB)" but no story covers this DB seeding requirement.

---

## 3. Epic Coverage Validation

### 3.1 Coverage Matrix

> Because the PRD uses feature tables (not numbered FRs), coverage is assessed by mapping PRD feature areas to epic domains.

| PRD Feature Area | Epic(s) Covering | Coverage Status |
|---|---|---|
| 3.1 Opportunity Discovery & Intelligence (11 features) | E05 (ingestion), E06 (discovery UI), E12 (forecasting) | ✅ Covered |
| 3.2 Document Analysis & Intelligence (4 features) | E06 (upload/parser), E07 (checklist, risk flag) | ✅ Covered |
| 3.3 Proposal Generation & Optimization (10 features) | E07 (all proposal features) | ✅ Covered |
| 3.4 Proposal Collaboration & Workflow (9 features) | E10 (collaboration, tasks, approvals) | ✅ Covered |
| 3.5 EU Grant Specialization (5 features) | E11 (grant tools) | ✅ Covered |
| 3.6 Compliance & Regulatory Intelligence (6 features) | E11 (ESPD, frameworks, audit), E02 (audit trail) | ✅ Covered |
| 3.7 Notifications, Alerts & Calendar (7 features) | E09 (full notifications epic) | ✅ Covered |
| 3.8 Analytics & Reporting (7 features) | E12 (all dashboards and reports) | ✅ Covered |
| 3.9 Subscription & Billing (8 features) | E08 (full billing epic) | ✅ Covered |
| 3.10 Platform Administration (7 features) | E12 (admin platform, Enterprise API) | ✅ Covered |
| **NFR1 Availability (99.5%)** | E01 (infra), E12 (load testing) | ⚠️ Partial — no explicit monitoring setup story |
| **NFR2 API latency (p95 < 200ms)** | E12 (S12.17 load testing) | ⚠️ Partial — tested at end, no interim performance gates |
| **NFR3 Data residency (EU)** | E01 (Terraform), Architecture | ⚠️ Partial — Terraform is placeholder in E01 |
| **NFR4 Security (JWT, TLS, AES)** | E02 (auth), E04 (AI gateway), E12 (S12.17 security audit) | ✅ Covered |
| **NFR5 Scalability (10K+ tenders)** | E12 (S12.17) | ⚠️ Partial — load testing deferred to Sprint 13–14 |
| **NFR6 GDPR** | E02 (audit), E12 (S12.17) | ❌ **GAP — No dedicated GDPR story** (right to erasure, DPA, consent) |
| **NFR7 Agent quality** | E04 (gateway logging), E12 (platform analytics) | ⚠️ Partial — KraftData eval runs not explicitly covered |
| **NFR8 i18n (BG+EN)** | E03 (S03.07 next-intl) | ✅ Covered |

### 3.2 Missing Requirements

#### ❌ Critical Missing: GDPR Right to Erasure

- **PRD NFR6:** "Right to erasure, DPAs with KraftData, data processing records"
- **Epic Coverage:** No story across any of the 12 epics implements a user data deletion flow, a "right to be forgotten" API, or a consent/data processing record mechanism.
- **Impact:** Regulatory compliance failure on launch. GDPR erasure is not optional.
- **Recommendation:** Add a story to E02 or E12 covering: user data deletion API, cascade delete across all schemas, audit log entry for deletion, notification of KraftData for AI data removal.

#### ❌ Critical Missing: Prometheus/Grafana Monitoring Setup

- **PRD NFR1/NFR2:** Grafana uptime monitoring and Prometheus histograms are cited as measurement mechanisms.
- **Epic Coverage:** E01 sets up infrastructure but no story explicitly provisions Prometheus, Grafana, Loki, or Jaeger. E12 (S12.17) references load testing but assumes monitoring tooling exists.
- **Recommendation:** Add a monitoring setup story to E01 Sprint 1–2 scope.

#### ⚠️ Warning: Demo-Milestone Billing Simulation

- **PRD Section 6.1:** "Billing is simulated (tier set manually in DB)" for the Sprint 8 demo.
- **Epic Coverage:** E08 is scheduled for Sprints 9–10, after the Sprint 8 demo. No story seeds or simulates tier data for the demo.
- **Recommendation:** Add a story to E02 or E06 that seeds demo tiers in the database (or documents a manual procedure).

#### ⚠️ Warning: KraftData Agent Quality Monitoring (NFR7)

- **PRD NFR7:** "KraftData eval-runs for continuous quality monitoring" with "eval score dashboards."
- **Epic Coverage:** E04 logs agent executions, but no epic defines eval dashboards or integrates with KraftData eval infrastructure.
- **Recommendation:** Add a story to E04 or E12 for agent evaluation result ingestion and display.

### 3.3 Coverage Statistics

| Metric | Count |
|---|---|
| Total PRD feature areas | 10 |
| Feature areas with full epic coverage | 10 |
| Total features within those areas | 74 |
| Total NFRs | 8 |
| NFRs fully covered | 3 |
| NFRs partially covered | 4 |
| NFRs with critical gap | 1 (GDPR) |
| **Overall feature-area coverage** | **100%** |
| **NFR coverage (fully)** | **37.5%** |
| **NFR coverage (partial + full)** | **87.5%** |

---

## 4. UX Alignment Assessment

### 4.1 UX Document Status

✅ **Found — Two UX documents:**
- `EU_Solicit_UX_Supplement_v1.md` — Design system, screen specifications, components, accessibility, mobile patterns, analytics dashboard
- `EU_Solicit_User_Journeys_and_Workflows_v1.md` — Personas, user flows, end-to-end dataflows

### 4.2 UX ↔ PRD Alignment

| UX Element | PRD Alignment | Status |
|---|---|---|
| 5 Personas (Maria, Georgi, Elena, Nikolai, Internal Admin) | PRD Section 2 defines identical 5 personas | ✅ Aligned |
| Opportunity listing with tier gating | PRD 3.1 — tiered access, limited free view | ✅ Aligned |
| AI summary panel with streaming | PRD 3.1 — SSE streaming, Starter+ | ✅ Aligned |
| Proposal workspace with Tiptap | PRD 3.3 — rich text editor, versioning | ✅ Aligned |
| Collaboration sidebar (comments, locks) | PRD 3.4 — section locking, threaded comments | ✅ Aligned |
| Subscription/upgrade flow | PRD 3.9 — Stripe Checkout, tier comparison | ✅ Aligned |
| Admin dashboard | PRD 3.10 — admin platform | ✅ Aligned |
| Analytics dashboards | PRD 3.8 — all dashboard types | ✅ Aligned |
| Accessibility (WCAG 2.1 AA) | PRD does not explicitly cite WCAG | ⚠️ UX over-specifies vs PRD |
| Mobile-first responsive | PRD does not specify mobile support level | ⚠️ UX over-specifies vs PRD |

### 4.3 UX ↔ Architecture Alignment

| UX Requirement | Architecture Support | Status |
|---|---|---|
| SSE streaming for AI (< 30s summary) | AI Gateway (S04.05) + Client API SSE proxy | ✅ Supported |
| Tiptap rich text editor (frontend) | Next.js 14 + React 18 frontend | ✅ Supported |
| Real-time section locking | Redis-based locking via Client API | ✅ Supported |
| PDF/DOCX export | Dedicated export endpoint in Client API (E07) | ✅ Supported |
| Google/Outlook calendar sync | Notification Service (E09) | ✅ Supported |
| iCal feed | Client API endpoint (E09) | ✅ Supported |
| Subdomain white-labelling (Enterprise) | Admin API + Nginx config (E12) | ⚠️ DNS cert provisioning unspecified |
| Mobile responsive UI (UX Supplement) | Next.js 14 + Tailwind responsive classes | ✅ Supported |
| Analytics materialized views | PostgreSQL + scheduled Celery refresh (E12) | ✅ Supported |
| Internationalization BG + EN | next-intl in E03 | ✅ Supported |

### 4.4 UX Alignment Warnings

⚠️ **White-label subdomain TLS provisioning not architected:** The UX and PRD both specify custom subdomains for Enterprise white-labelling but neither the Architecture nor the Epics specify how SSL/TLS certificates are provisioned per subdomain (cert-manager, Let's Encrypt, wildcard cert, manual). This is a deployment-time complexity that could block Enterprise delivery.

⚠️ **No skeleton/loading screen specifications in epics:** The UX Supplement specifies skeleton loading patterns, but only E03 (S03.10) mentions loading states generically. Individual feature epics (E06, E07, E10) do not reference specific loading state requirements per page.

---

## 5. Epic Quality Review

### 5.1 Epic Summary Table

| Epic | Title | Stories | Points | Sprint | Milestone | User Value | Forward Deps |
|---|---|---|---|---|---|---|---|
| E01 | Infrastructure & Monorepo Foundation | 10 | 34 | 1–2 | Demo | ⚠️ Technical | None |
| E02 | Authentication & Identity | 12 | 34 | 1–2 | Demo | ✅ Yes | E03, E06, E08, E10 |
| E03 | Frontend Shell & Design System | 12 | 37 | 1–2 | Demo | ✅ Yes | E06, E07, E08, E09, E10 |
| E04 | AI Gateway Service | 10 | 34 | 3–4 | Demo | ⚠️ Technical | E05, E06, E07, E11 |
| E05 | Data Pipeline & Opportunity Ingestion | 12 | 34 | 3–4 | Demo | ✅ Yes | E06, E09 |
| E06 | Opportunity Discovery & Intelligence | 14 | 55 | 5–6 | Demo | ✅ Yes | E07, E08 |
| E07 | Proposal Generation & Document Intelligence | 16 | 55 | 7–8 | Demo | ✅ Yes | E08, E10 |
| E08 | Subscription & Billing | 14 | 55 | 9–10 | Beta | ✅ Yes | E09, E12 |
| E09 | Notifications, Alerts & Calendar | 14 | 55 | 9–10 | Beta | ✅ Yes | E10, E12 |
| E10 | Collaboration, Tasks & Approvals | 16 | 55 | 11–12 | MVP | ✅ Yes | E12 |
| E11 | EU Grant Specialization & Compliance | 16 | 55 | 11–12 | MVP | ✅ Yes | None |
| E12 | Analytics, Reporting & Admin Platform | 18 | 55 | 13–14 | MVP | ✅ Yes | None |
| **TOTAL** | | **154** | **558** | | | | |

### 5.2 Best Practices Compliance Checklist

| Best Practice | E01 | E02 | E03 | E04 | E05 | E06 | E07 | E08 | E09 | E10 | E11 | E12 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Delivers user value | ⚠️ | ✅ | ✅ | ⚠️ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Can function independently | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Stories appropriately sized | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| No forward dependencies | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| DB tables created when needed | ⚠️ | ✅ | N/A | ✅ | ⚠️ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | N/A |
| Clear acceptance criteria | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Given/When/Then AC format | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| FR traceability to PRD | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

### 5.3 Critical Violations

#### 🔴 CRITICAL — E01 and E04 are Infrastructure-Only Epics with No Direct User Value

**E01 — Infrastructure & Monorepo Foundation:**
- Stories S01.06–S01.10 (shared packages, CI pipeline, Helm, Terraform) deliver zero user-facing functionality.
- Epic goal states "establish monorepo structure, local development environment" — these are technical milestones, not user outcomes.
- **Verdict:** E01 is a valid **foundational sprint** but should be labelled as "Foundation Sprint" rather than positioned as a user-value epic. It is acceptable as Sprint 1–2 because nothing else can start without it, but it should not be sequenced alongside user-value epics as if equivalent.
- **Recommended action:** Re-label E01 as "Foundation Sprint — Infrastructure" and explicitly acknowledge it is a technical prerequisite, not a user epic. No stories need to be removed.

**E04 — AI Gateway Service:**
- The entire epic is an internal-only service. No story delivers a user-visible feature.
- E04 is correctly positioned (Sprint 3–4, before E06 which uses it), but the epic description does not acknowledge it is a technical enabler rather than user-value delivery.
- **Recommended action:** Label E04 as "Enabler Epic — AI Platform Integration." Accept as valid technical dependency pattern. The sequencing is correct.

#### 🔴 CRITICAL — No FR Traceability Across All 12 Epics

**Across all 12 epics, zero stories reference a PRD requirement by identifier.**

This is the most significant planning deficiency found. Without traceability:
- There is no formal way to verify that all 74 PRD features have implementation coverage
- A developer picking up a story has no way to reference the originating requirement
- Regression testing cannot be tied back to requirement origin
- Scope changes cannot be impact-assessed against known requirements

**Root cause:** The PRD itself does not assign numbered identifiers (FR1, FR2…) to its features, making cross-referencing require agreement on a numbering convention.

**Recommended action (High Priority):**
1. Assign FR identifiers to all 74 PRD features (FR-001 through FR-074, grouped by section 3.1–3.10)
2. Add a `PRD References:` field to each epic's header listing covered FRs
3. Add FR traceability comments to individual story acceptance criteria where appropriate

#### 🔴 CRITICAL — No Given/When/Then Acceptance Criteria Format

**All 154 stories across all 12 epics use checkbox/declarative format, not Given/When/Then (BDD).**

Example of what exists (S02.02):
```
[ ] A new company + admin user can be created via POST /auth/register
[ ] Email is unique at DB level (constraint + API error)
```

This format is readable but:
- Not testable in an automated BDD framework (Cucumber, Behave, SpecFlow)
- Does not specify actor context (Given the user is unauthenticated, When...)
- Does not specify system state preconditions
- Omits error/edge-case conditions in many stories

**Impact:** QA automation will require significant rework of ACs before tests can be written.

**Recommended action:** This is a systemic issue. Rather than rewriting all 154 stories before implementation, the team should agree to either:
1. Accept declarative ACs as-is and write BDD tests in a separate test epic/sprint
2. Convert ACs to G/W/T format as stories enter the sprint backlog (Definition of Ready gate)

### 5.4 Major Issues

#### 🟠 MAJOR — E01: All Database Schemas Designed Upfront in Sprint 1–2

**S01.03 (PostgreSQL Schema Design)** creates all 6 schemas (client, pipeline, admin, notification, gateway, shared) with role-based access in the very first sprint. While individual tables are created per-story via Alembic migrations, the upfront "schema design" story implies designing all table structures in Sprint 1–2 before downstream epics have iterated their requirements.

If any later epic changes a schema, Sprint 1–2 artefacts must be retroactively updated.

**Recommendation:** S01.03 should create empty schemas and role grants only. Table column designs remain in per-story Alembic migrations.

Similarly, **E05 S05.01** creates all 4 pipeline tables upfront in a single schema migration story.

#### 🟠 MAJOR — E04: Circuit Breaker State is Per-Instance (Won't Scale)

**S04.06 (Circuit Breaker and Retry Logic):** Circuit breaker state is in-memory per service instance. If AI Gateway scales to multiple replicas (HPA 2–10 pods), each pod tracks its own failure counts independently. A KraftData API outage might only trip the circuit breaker in one pod while others continue sending requests, defeating the purpose.

- **Impact:** Production resilience failure in scaled deployments.
- **Recommendation:** Store circuit breaker state in Redis (shared across instances), keyed per-agent.

#### 🟠 MAJOR — E06: Tier Enforcement Point Undefined

**S06.02 (Tier-Gated Response Serialization)** and **S06.03 (Usage Metering Middleware)** implement tier access but:
- The enforcement point (query-time filtering vs. response serialization) is unspecified
- Free-tier opportunity IDs not listed in field allowlist (needed for linking to detail pages)
- "Billing period" for metering not defined (UTC calendar month vs. subscription anniversary)

**Impact:** Incorrect tier enforcement could expose paid fields to free users, or block free users from navigating to opportunities.

#### 🟠 MAJOR — E08: Trial Abuse Prevention Not Specified

**S08.03 (14-Day Professional Trial):** The epic states "one trial per company" but provides no mechanism to prevent abuse via new company account creation.

**Impact:** Revenue leakage from trial farming.

#### 🟠 MAJOR — E09: Event Publishing Source Ambiguity (Trial Expiry)

**S09.11 (Trial Expiry & Stripe Usage Sync):** Assumes a `subscription.trial_expiring` event arrives on the Redis Stream, but E08 (S08.05) handles trial expiry via Stripe webhooks (`customer.subscription.trial_will_end`). No epic explicitly publishes this event to Redis.

**Impact:** Integration failure — the Notification Service will not receive trial expiry events.

#### 🟠 MAJOR — E10: Section Lock Auto-Extension Missing

**S10.03 (Section Locking API):** Locks expire after 15 minutes with no TTL heartbeat while the user is actively editing. Long-form proposal writing will silently lose locks mid-session.

**Impact:** Data loss or overwrite in collaborative editing scenarios.

#### 🟠 MAJOR — E12: Pipeline Forecasting Agent Undefined

**S12.07 (Pipeline Forecasting Dashboard):** References a "Pipeline Forecasting Agent" but this agent is not defined in E04's agent registry or E11's agent list. The PRD (3.1, 3.8) cites this feature as Professional+.

**Impact:** Sprint 13–14 story blocks on an unbuilt KraftData agent dependency.

### 5.5 Minor Concerns

#### 🟡 Scattered Tier Enforcement Logic
Tier gating logic is implemented independently in E06 (discovery), E07 (proposals), E08 (billing), and E12 (analytics). No unified tier policy layer exists. As tier rules evolve, changes must be made in multiple epics simultaneously.
- **Recommendation:** Document a "Tier Policy" ADR; designate enforcement location in Client API middleware.

#### 🟡 No Redis Event Catalog
Redis Streams used across E01, E05, E08, E09, E10 with no central contract document defining event schemas, publishers, consumers, and retention.
- **Recommendation:** Create a lightweight event catalog document before Sprint 3.

#### 🟡 Terraform Scaffolding Never Completed
E01 S01.10 creates "Terraform Scaffold & Placeholder Modules." No subsequent epic converts placeholders into working infrastructure-as-code for production.
- **Recommendation:** Add Terraform completion story to E12's pre-production sprint.

#### 🟡 AI Agent Error Handling Standardised Too Late
E11 S11.16 is the first story addressing "Agent Error Handling Hardening," but AI agents are called from E06 onward. Four epics (E06, E07, E09, E10) invoke agents without a standardised error handling pattern.
- **Recommendation:** Move agent error handling standards to E04 S04.10 (Integration Tests), establishing the pattern early.

#### 🟡 E11: ESPD JSONB Schema Undefined
S11.01 and S11.02 reference "structured data mapped to ESPD XML schema" but no JSONB field mapping or example is provided. Developers must independently interpret EU ESPD specification.

#### 🟡 E12: Materialized View Refresh Timing Unspecified
S12.01 says "daily" refresh with no time of day, timezone, or `REFRESH CONCURRENTLY` index requirements specified. Midday refresh could degrade analytics query performance.

#### 🟡 White-Label SSL Provisioning Unspecified
Custom subdomains (E12 S12.12) require per-subdomain TLS certificates. No mechanism specified in epics or architecture (cert-manager, Let's Encrypt wildcard, manual provisioning).

---

## 6. Summary and Recommendations

### 6.1 Overall Readiness Status

> ## ⚠️ NEEDS WORK

The planning artifacts are comprehensive and thoughtfully structured. Epic scope is correct, story breakdowns are detailed, the architecture is sound, and UX documentation is thorough. However, several systemic deficiencies must be addressed before development begins.

### 6.2 Critical Issues Requiring Immediate Action

| # | Issue | Severity | Epic(s) | Action |
|---|---|---|---|---|
| 1 | No GDPR Right to Erasure story exists anywhere | 🔴 Critical | E02 or E12 | Add story: data deletion API, cascade delete, KraftData notification |
| 2 | No FR traceability — zero epic stories reference PRD requirements | 🔴 Critical | All 12 | Assign FR-001…FR-074 to PRD; add FR headers to epics |
| 3 | No Given/When/Then acceptance criteria across all 154 stories | 🔴 Critical | All 12 | Agree on AC conversion strategy (Definition of Ready gate) |
| 4 | Prometheus/Grafana setup story missing from E01 | 🔴 Critical | E01 | Add monitoring setup story to Sprint 1–2 |
| 5 | Circuit breaker state not shared across AI Gateway replicas | 🟠 Major | E04 S04.06 | Use Redis for circuit breaker state per agent |
| 6 | Pipeline Forecasting Agent not provisioned in any epic | 🟠 Major | E04, E12 | Define agent in E04 registry; add provisioning story |
| 7 | Event publishing source ambiguous for trial expiry | 🟠 Major | E08, E09 | Create event catalog; designate E08 as publisher |
| 8 | Section lock auto-extension heartbeat missing | 🟠 Major | E10 S10.03 | Add TTL heartbeat endpoint to extend lock on active edit |
| 9 | Demo-milestone tier seeding unspecified (Sprint 8 demo) | 🟠 Major | E02 or E06 | Add DB seed story or documented procedure |
| 10 | Tier enforcement point undefined in E06 | 🟠 Major | E06 S06.02 | Specify enforcement layer; define billing period |

### 6.3 Recommended Next Steps

**Before Sprint 1 starts:**
1. Assign FR identifiers (FR-001 to FR-074) to all PRD features
2. Add FR reference mapping to each epic header
3. Add GDPR right-to-erasure story to E02 or E12
4. Add Prometheus/Grafana setup story to E01
5. Create Redis Event Catalog document
6. Define "billing period" (document in architecture ADR)

**Before Sprint 3 (AI Gateway & Data Pipeline):**
7. Fix E04 circuit breaker to use Redis shared state
8. Define Pipeline Forecasting Agent in E04 agent registry
9. Designate E08 as publisher of `subscription.trial_expiring` event

**Before Sprint 5 (Opportunity Discovery):**
10. Resolve tier enforcement point and billing period definition in E06
11. Document white-label TLS provisioning strategy

**Ongoing (Definition of Ready gate):**
12. Convert declarative ACs to Given/When/Then as stories enter sprint
13. Apply agent error handling standards from E06 onward (not wait until E11)

**Before Sprint 9 (Billing & Collaboration):**
14. Address trial abuse prevention mechanism in E08
15. Add section lock heartbeat to E10 S10.03

### 6.4 What Is Ready — Strengths to Preserve

- ✅ **Epic scope and domain coverage** — all 10 PRD feature areas are mapped to epics
- ✅ **Story sizing** — 154 stories is well-calibrated; no epic-sized stories or micro-tasks
- ✅ **Epic sequencing** — sprint ordering respects dependencies; no forward dependencies found
- ✅ **Architecture alignment** — 5 services cleanly map to epic domains; no architectural gaps for core features
- ✅ **UX alignment** — UX supplement and user journeys are coherent with PRD and architecture
- ✅ **Tier gating coverage** — all 74 features correctly identify target tiers
- ✅ **Milestone-driven delivery** — Demo / Beta / MVP cadence provides clear checkpoints

### 6.5 Issue Count Summary

| Severity | Count |
|---|---|
| 🔴 Critical | 4 |
| 🟠 Major | 7 |
| 🟡 Minor | 7 |
| **Total** | **18** |

This assessment identified **18 issues** across **6 categories**: FR traceability, GDPR compliance, acceptance criteria quality, monitoring, inter-service integration, and implementation detail gaps. Address the 4 critical issues before Sprint 1 begins. The 7 major issues should be resolved before the affected epics enter the sprint. These findings can be used to improve the artifacts — the planning foundation is strong and these are correctable gaps, not fundamental rework.

---

*Report generated: 2026-04-09 | Assessment workflow: bmad-check-implementation-readiness | Project: EU Solicit*
