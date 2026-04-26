---
title: 'TEA Test Design → BMAD Handoff Document'
version: '1.4'
workflowType: 'testarch-test-design-handoff'
inputDocuments:
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
sourceWorkflow: 'testarch-test-design'
generatedBy: 'TEA Master Test Architect'
generatedAt: '2026-04-19'
projectName: 'EU Solicit'
---

# TEA → BMAD Integration Handoff

## Purpose

This document bridges TEA's test design outputs with BMAD's epic/story decomposition workflow (`create-epics-and-stories`). It provides structured integration guidance so that quality requirements, risk assessments, and test strategies flow into implementation planning.

**Version history:**
- v1.0 (2026-04-05): Initial system-level handoff
- v1.1 (2026-04-07): R-003 (SSE saturation) and R-006 (entity RBAC) elevated to HIGH; agent count corrected
- v1.2 (2026-04-09): Agent count corrected to 29 (per Architecture v4 §11.2); added R-016 (locale routing, score 4) and R-017 (JWT refresh race, score 4); risk register expanded to 17; coverage updated to 92 tests; data-testid requirements added; pre-implementation blocker TB-04 (Redis DLQ) added
- v1.3 (2026-04-09): Added R-018 (ESPD XML conformance, score 6) from E11 epic-level analysis; risk register expanded to 18; coverage updated to 94 tests; E11 quality gate updated; R-018 added to risk-to-story mapping
- v1.4 (2026-04-19): Synchronized with test-design v2 documents; corrected critical architecture references (Kafka → Redis 7 Streams; Elasticsearch → PostgreSQL FTS; "AI Solicit Service" → KraftData AI Gateway 29 agents); system-level scenario plan expanded to 32 canonical scenarios (P0×8 / P1×12 / P2×8 / P3×4); added Python/eusolicit-test-utils code examples; testability concerns section refreshed with TB-01–TB-04 blockers aligned to current codebase; architectural gap list updated

## TEA Artifacts Inventory

| Artifact | Path | BMAD Integration Point |
|----------|------|----------------------|
| Architecture Test Design | `eusolicit-docs/test-artifacts/test-design-architecture.md` | Epic quality requirements, testability blockers |
| QA Test Design | `eusolicit-docs/test-artifacts/test-design-qa.md` | Story acceptance criteria, test coverage plan |
| Risk Assessment | (embedded in both documents) | Epic risk classification, story priority |
| Coverage Strategy | (embedded in QA document) | Story test requirements, P0–P3 mapping |

## Epic-Level Integration Guidance

### Risk References

The following P0/P1 risks should appear as epic-level quality gates:

| Risk ID | Category | Score | Epic Impact | Quality Gate |
|---------|----------|-------|-------------|-------------|
| R-001 | TECH | 9 | E04 (AI Gateway), E06 (Opportunity Discovery), E07 (Proposal Generation) | AI Gateway mock mode operational before AI feature epics begin |
| R-002 | SEC | 6 | E02 (Authentication/Identity), all data-accessing epics | Cross-tenant isolation tests pass before any user-facing epic exits QA |
| R-003 | PERF | 6 | E04 (AI Gateway) | SSE connection pool isolation validated before load testing; HPA metric on SSE count |
| R-004 | BUS | 6 | E08 (Subscription/Billing) | Full billing lifecycle test suite (trial, upgrade, downgrade, add-on, VAT) passes before Beta |
| R-006 | SEC | 6 | E02 (Authentication/Identity), E10 (Collaboration) | Entity-level RBAC permission matrix validated before Collaboration epic |
| R-018 | TECH/BUS | 6 | E11 (EU Grant Specialization & Compliance) | ESPD XML serialisation + XSD v2.1+ validation layer deployed before E11 ESPD export tested; EU ESPD XSD pinned in Docker image at build time |

### Quality Gates

| Epic | Gate Criteria | Risk Reference |
|------|--------------|---------------|
| E01 (Infrastructure) | Docker Compose test environment operational; test data seeding API deployed | TB-01 |
| E02 (Auth/Identity) | JWT validation tests pass; cross-tenant isolation validated; entity-level RBAC matrix tested | R-002, R-006 |
| E03 (Frontend Shell) | i18n framework operational; tier-gated UI components render correctly for all tiers; locale routing middleware tested (BG/EN); JWT refresh deduplication tested | R-012, R-016, R-017 |
| E04 (AI Gateway) | Mock mode operational; circuit breaker tested; SSE proxy functional; SSE connection pool isolated | R-001, R-003 |
| E05 (Data Pipeline) | Crawl → normalize → upsert flow tested; stale-data graceful degradation verified | R-005 |
| E06 (Opportunity Discovery) | Search/filter tests pass; tier-gated access validated across all 4 tiers | R-012 |
| E07 (Proposal Generation) | AI draft generation via mock; proposal versioning; export (PDF/DOCX) | R-001 |
| E08 (Subscription/Billing) | Full Stripe lifecycle: trial → paid → cancel; add-ons; VAT; test clocks validated | R-004 |
| E09 (Notifications/Alerts) | Email digest delivery; Redis Streams event flow validated across all 7 streams | R-007 |
| E10 (Collaboration) | Section locking + TTL; per-proposal roles; approval workflows | R-006, R-015 |
| E11 (Grants/Compliance) | Compliance framework CRUD; ESPD auto-fill; ESPD XML validates against EU XSD v2.1+; framework-specific validation; AI Gateway mock covers all 8 E11 agent types; agent error handling gate (timeout, retry, 503 with graceful message) | R-001, R-018 |
| E12 (Analytics/Admin) | Admin VPN access; platform analytics; audit log viewer | — |

## Story-Level Integration Guidance

### P0/P1 Test Scenarios → Story Acceptance Criteria

The following critical test scenarios MUST be acceptance criteria in their respective stories:

| Test ID | Test Scenario | Recommended Story/Epic | Acceptance Criteria Text |
|---------|--------------|----------------------|------------------------|
| P0-001 | User registration creates company + subscription | E02 Auth | "When a user registers, a company record AND a Professional trial subscription are created" |
| P0-003 | Expired JWT returns 401 | E02 Auth | "When an expired JWT is presented, the API returns 401 Unauthorized" |
| P0-007 | Cross-tenant data isolation | E02 Auth | "A user from Company A cannot access any data belonging to Company B" |
| P0-008 | Free-tier limited view | E06 Opportunity Discovery | "Free-tier users see only name, deadline, location, type, and status" |
| P0-011 | Trial lifecycle | E08 Billing | "After 14 days, trial users are downgraded to Free tier with data preserved" |
| P0-012 | Usage metering enforcement | E08 Billing | "When a Starter user exhausts 10/10 AI summaries, the next request returns 429 with upgrade prompt" |
| P0-013 | AI summary streaming | E04 AI Gateway + E06 Discovery | "AI summary is delivered via SSE stream; usage meter increments by 1" |
| P0-016 | Document upload + virus scan | E05 Data Pipeline | "Uploaded files are scanned by ClamAV; infected files are rejected with 422" |
| P1-010 | Section locking | E10 Collaboration | "When User A locks a section, User B receives 423 Locked; lock expires after 15 min inactivity" |
| P1-013 | Approval workflow | E10 Collaboration | "Proposals progress through configurable stages; each stage requires sign-off from the designated role" |

### Data-TestId Requirements

For E2E testability, the following `data-testid` attributes should be specified in frontend stories:

| Component | Recommended data-testid | Used By Test |
|-----------|------------------------|-------------|
| Opportunity list item | `opportunity-card-{id}` | P0-017, P1-001 |
| Tier upgrade prompt | `tier-upgrade-cta` | P0-008, P0-012 |
| AI summary panel | `ai-summary-panel` | P0-013 |
| Usage counter | `usage-counter-{feature}` | P0-012, P2-015 |
| Proposal editor | `proposal-editor` | P1-007 |
| Section lock indicator | `section-lock-{key}` | P1-010 |
| Login form | `login-form`, `login-email`, `login-password`, `login-submit` | P0-002 |
| Registration form | `register-form`, `register-email`, `register-password`, `register-company` | P0-001 |
| Alert preferences | `alert-cpv-picker`, `alert-region-picker`, `alert-frequency` | P1-004 |

## Risk-to-Story Mapping

| Risk ID | Category | P×I | Recommended Story/Epic | Test Level |
|---------|----------|-----|----------------------|-----------|
| R-001 | TECH | 9 | E04: AI Gateway circuit breaker + mock mode | API + E2E |
| R-002 | SEC | 6 | E02: Cross-tenant isolation middleware | API |
| R-003 | PERF | 6 | E04: SSE connection pool isolation + HPA metric | Perf (k6) |
| R-004 | BUS | 6 | E08: Full billing lifecycle (trial, upgrade, add-on, VAT) | API |
| R-005 | DATA | 4 | E05: Data pipeline graceful degradation | API |
| R-006 | SEC | 6 | E02: Entity-level RBAC enforcement | API |
| R-007 | TECH | 4 | E09: Redis Streams dead letter handling (7 streams) | API |
| R-008 | PERF | 4 | E01: PostgreSQL index optimization | Perf (k6) |
| R-009 | OPS | 4 | E09: Calendar sync OAuth token management | API |
| R-010 | SEC | 3 | E05: ClamAV integration for uploads | API |
| R-012 | BUS | 4 | E06: Tier gating parametrized tests | API |
| R-015 | SEC | 4 | E10: Section locking race conditions | API |
| **R-016** ★NEW | TECH | 4 | E03: Frontend Shell locale middleware routing (BG/EN) | E2E |
| **R-017** ★NEW | SEC | 4 | E02/E03: JWT refresh race condition (concurrent browser tabs) | E2E |
| **R-018** ★NEW | TECH/BUS | 6 | E11: ESPD XML serialisation + EU ESPD XSD v2.1+ validation endpoint | API |

## Recommended BMAD → TEA Workflow Sequence

1. **TEA Test Design** (`TD`) → produces this handoff document ✅ (current)
2. **BMAD Create Epics & Stories** → consumes this handoff, embeds quality requirements
3. **TEA ATDD** (`AT`) → generates acceptance tests per story
4. **BMAD Implementation** → developers implement with test-first guidance
5. **TEA Automate** (`TA`) → generates full test suite
6. **TEA Trace** (`TR`) → validates coverage completeness

## Phase Transition Quality Gates

| From Phase | To Phase | Gate Criteria |
|-----------|----------|--------------|
| Test Design | Epic/Story Creation | All 6 high-priority risks (R-001 through R-006, R-018) have assigned owners + mitigation strategy; TB-01 through TB-04 scheduled in sprint plan |
| Epic/Story Creation | ATDD | Stories have P0/P1 acceptance criteria embedded; data-testid attributes specified in frontend stories; pre-implementation blocker stories (TB-01–TB-04) assigned to sprints |
| ATDD | Implementation | Failing acceptance tests exist for all P0/P1 scenarios; CI pipeline runs and fails (red phase confirmed) |
| Implementation | Test Automation | All acceptance tests pass (green phase); no open P0/P1 bugs; cross-tenant isolation sweep = 0 leaks |
| Test Automation | Release | Traceability matrix shows ≥80% coverage of P0/P1 requirements; all Stripe lifecycle tests pass; performance baselines met (REST p95 < 200ms; SSE TTFB < 500ms) |

---

## Pre-Implementation Blockers (BMAD Must Schedule)

| Blocker ID | Title | Blocks Epics | Required Sprint | Owner |
|-----------|-------|-------------|----------------|-------|
| **TB-01** | Test data seeding API (`POST /api/test-data/seed` + cleanup) | E02–E12 | Sprint 3 | Backend Lead |
| **TB-02** | KraftData mock service (all 29 entity types + SSE streaming; includes 8 E11 agent types: Grant Eligibility, Budget Builder, Logframe Generator, Consortium Finder, Reporting Template, ESPD Auto-Fill, Framework Suggestion, Regulation Tracker) | E04, E05, E06, E07, E10, E11 | Sprint 3 | Backend Lead |
| **TB-03** | Stripe test mode + Stripe CLI webhook forwarding in CI | E08 | Sprint 4 | Backend Lead |
| **TB-04** | Redis Streams DLQ (per consumer group + Prometheus monitoring) | E05, E09 | Sprint 5 | Backend Lead |
