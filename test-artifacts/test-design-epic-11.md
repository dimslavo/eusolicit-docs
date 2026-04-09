---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-09'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 11
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-11-grants-compliance.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'resources/knowledge/risk-governance.md'
  - 'resources/knowledge/probability-impact.md'
  - 'resources/knowledge/test-levels-framework.md'
  - 'resources/knowledge/test-priorities-matrix.md'
---

# Test Design: Epic 11 — EU Grant Specialization & Compliance

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**Status:** Draft
**Sprint:** 11–12 | **Points:** 55 | **Dependencies:** E04, E06, E07 | **Milestone:** MVP

---

## Executive Summary

**Scope:** Epic-level test design for E11 — EU grant application toolkit (Grant Eligibility, Budget Builder, Consortium Finder, Logframe Generator, Reporting Template Generator agents), ESPD profile management with auto-fill and XML/PDF export, and platform-wide compliance administration (framework CRUD, opportunity assignment, auto-suggestion via Framework Suggestion Agent, Regulation Tracker Agent on Celery Beat schedule). All agent invocations flow through the AI Gateway (E04). This epic introduces **8 new KraftData agent types**, making it the most AI-intensive epic to date.

**Risk Summary:**

- Total risks identified: 12
- High-priority risks (score ≥ 6): 5
- Critical categories: TECH (AI Gateway error handling), DATA (ESPD XML conformance), SEC (admin authorization, ESPD RLS), BUS (budget arithmetic)

**Coverage Summary:**

- P0 scenarios: 10 (~20–35 hours)
- P1 scenarios: 18 (~20–35 hours)
- P2 scenarios: 20 (~15–25 hours)
- P3 scenarios: 8 (~10–20 hours)
- **Total effort:** ~65–115 hours (~2–3.5 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **KraftData agent output quality (AI eval)** | Owned by KraftData; EU Solicit owns orchestration and parsing only | All agent interactions tested via TB-02 deterministic mock responses |
| **EU ESPD legal compliance review** | ESPD XML must be structurally valid per XSD schema; legal accuracy of completed fields is a business/legal concern | XML structure validated against official EU ESPD XSD in P1 tests |
| **ESPD rendering in third-party portals** | EU Solicit generates valid XML; portal-side compatibility is out of QA scope | EU ESPD XSD validation ensures structural correctness |
| **Stripe billing flows** | Covered in E06 test design | Billing state assumed valid via TB-01 seeding |
| **Proposal generation and export** | Covered in E07 | S11.16 journey 3 reuses E07 compliance checker — E07 regression must be green |
| **Data pipeline crawl logic** | Covered in E04 | Framework Suggestion Agent receives opportunity metadata; crawl parsing not re-tested |
| **Calendar sync (Google/Microsoft)** | Covered in E08 | Not referenced by any E11 story |
| **Logframe/budget accuracy (AI output quality)** | KraftData responsibility; E11 tests parser correctness and structural completeness | Arithmetic validation tests catch parser-level inconsistencies (E11-R-004) |

---

## Risk Assessment

> **Note:** P0/P1/P2/P3 designations below indicate test priority and risk, not execution timing. See Execution Strategy for timing decisions.

### High-Priority Risks (Score ≥ 6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E11-R-001** | TECH | AI Gateway error handling across 8 new agent types — S11.16 hardened timeout (30s default) and retry logic (1 retry, exp backoff). If not consistently applied across all E11 agent-backed endpoints, transient failures return raw 500s; users lose in-progress grant work with no recovery path. Extends system R-001. | 2 | 3 | **6** | Extend TB-02 mock with configurable failure injection per agent type; test every E11 endpoint with forced timeout → structured 503; test retry flow | Backend Lead | Sprint 11 |
| **E11-R-002** | DATA | ESPD XML schema non-conformance — ESPD Auto-Fill export (S11.03) generates EU ESPD-compliant XML for Parts II-V. Invalid namespace, missing required elements, or malformed structure → ESPD rejected by EU procurement portals (TED, eTendering). Invalid ESPD = company misses bid deadline; high-stakes data loss scenario. | 2 | 3 | **6** | Obtain EU ESPD XSD (v2.1+); validate exported XML against schema in P1 tests; validate both auto-filled and manually completed profiles | Backend Lead | Sprint 11 |
| **E11-R-003** | SEC | Admin endpoint authorization gaps (S11.08, S11.09, S11.10) — compliance framework CRUD, framework assignment, auto-suggestion management, regulation tracker, and platform settings are admin-only. If admin-only middleware is absent or misconfigured, company users could create/modify compliance frameworks, override regulatory change status, or alter platform settings. Extends system R-006. | 2 | 3 | **6** | Test all S11.08–S11.10 endpoints with company JWT → verify 403; test with expired admin JWT → 401; verify admin paths are not accessible via company API route | Backend | Sprint 11 |
| **E11-R-004** | BUS | Budget arithmetic consistency — Budget Builder Agent (S11.05) returns a parsed budget object. If the parser does not validate that line items sum to totals, overhead_rate is correctly applied, and co-financing splits (EU contribution + own contribution = total_requested_funding) are arithmetically consistent, incorrect budget data is presented to users for EU grant applications. Incorrect budget = rejected grant application. | 2 | 3 | **6** | Test parser with valid response → arithmetic asserted correct; test with inconsistent AI response → validation error returned before serving to user; test per-partner breakdown summation | Backend Lead | Sprint 11 |
| **E11-R-005** | SEC | ESPD company-scoped RLS enforcement — ESPD profiles contain sensitive company declarations (exclusion grounds, criminal conviction declarations, financial standing). company_id FK + RLS must prevent cross-company access. Missing or misconfigured RLS allows Company A to read/modify Company B's ESPD declarations. Extends system R-002. | 2 | 3 | **6** | Create Company A + Company B with ESPD profiles; assert Company A JWT cannot GET/PATCH/DELETE Company B profile IDs (expect 404 to avoid enumeration); assert POST /espd-profiles binds to JWT company_id (not user-supplied) | Backend | Sprint 11 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E11-R-006 | OPS | Regulation Tracker Celery Beat reliability — if the scheduled task fails silently (Beat misconfiguration, agent timeout not retried, or DB write failure on regulatory_changes table), regulatory changes go undetected. Admins act on stale compliance data. | 2 | 2 | 4 | Test Celery task with mocked agent returning regulatory changes; test agent timeout scenario — task records error, does not crash; test DB write; verify manual trigger via `POST /admin/regulatory-changes/trigger` (or equivalent) | Backend Lead |
| E11-R-007 | DATA | Framework suggestion queue state atomicity — on suggestion acceptance, two writes must be atomic: `framework_suggestions.status = accepted` + `opportunity_compliance_frameworks` insert. Partial failure leaves opportunity without framework but suggestion marked accepted. | 2 | 2 | 4 | Test accept flow as DB transaction; simulate DB failure mid-accept → verify rollback (no orphaned accepted suggestion without corresponding assignment); test reject flow — dismissed suggestion does not auto-assign | Backend Lead |
| E11-R-008 | DATA | Logframe parser field completeness — Logframe Generator returns a complex nested structure (logical_framework, work_packages, gantt_data, deliverable_table). If parser silently drops optional but expected fields (e.g., gantt_data absent), frontend gets incomplete structure and renders incorrectly without error. | 2 | 2 | 4 | Test parser with complete agent response → all fields mapped; test with gantt_data absent → structured partial response with explicit null; test with missing work_packages → error or empty list, not crash | Backend Lead |
| E11-R-009 | BUS | Framework deletion guard correctness — S11.08 requires preventing deletion of frameworks currently assigned to active opportunities. If the guard checks the wrong table (inline FK vs join table), or ignores opportunity status (active vs archived), active opportunities can lose their compliance framework assignment mid-review. | 2 | 2 | 4 | Test delete framework assigned to active opportunity → 409 Conflict; test delete framework with archived-only opportunities → allowed; test hard-delete vs soft-delete path | Backend Lead |
| E11-R-010 | TECH | Hybrid national+EU compliance assignment edge cases — supporting multiple frameworks per opportunity (e.g., Bulgarian national procurement law + Horizon Europe rules) requires join-table semantics and correct enforcement at the compliance check layer. Mixed-framework validation rule conflicts are not handled by E11 (compliance checker is E07), but assignment correctness must be verified. | 2 | 2 | 4 | Test assigning 2 frameworks (national + EU) to same opportunity; test removing one without affecting the other; test GET /admin/opportunities/:id/compliance-frameworks returns both; test suggestion that conflicts with existing assignment | Backend |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E11-R-011 | PERF | DOCX export performance — Reporting Template Generator (S11.07) + Logframe Panel DOCX export (S11.12) use python-docx for potentially large reports. No concurrency cap documented. Concurrent DOCX generation may exhaust Celery workers. | 1 | 2 | 2 | Monitor; add k6 scenario in P3 with 10 concurrent DOCX requests; note worker memory limit in capacity planning |
| E11-R-012 | DATA | Consortium Finder result field completeness — agent returns ranked partner suggestions. Missing optional fields (contact_info, past_projects) should degrade gracefully (empty list/null), not crash the endpoint or UI. | 2 | 1 | 2 | Monitor; add P2 test for partial agent response (missing contact_info) → gracefully returns partner card without contact section |

### Risk Category Legend

- **TECH**: Architecture/integration (fragility, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data integrity (loss, corruption, inconsistency)
- **BUS**: Business impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

---

## Entry Criteria

- [ ] E04 (AI Gateway) stable; TB-02 mock mode extended for all 8 E11 agent types (Grant Eligibility, Budget Builder, Consortium Finder, Logframe Generator, Reporting Template Generator, ESPD Auto-Fill, Framework Suggestion, Regulation Tracker) with fixture responses + configurable failure injection
- [ ] E06 (billing) and E07 (compliance checker) stable — E11 depends on both per epic header
- [ ] S11.01 migrations merged; dev DB seeded with sample compliance frameworks and ESPD profiles
- [ ] TB-01 test data seeding extended to support: ESPD profiles (with structured espd_data), compliance frameworks, opportunity_compliance_frameworks join records, regulatory_changes records
- [ ] EU ESPD XSD obtained and available to test suite for XML conformance validation (E11-R-002)
- [ ] Celery Beat test mode (manual trigger or test-only endpoint) available for Regulation Tracker tests (S11.10)

## Exit Criteria

- [ ] All 10 P0 tests passing
- [ ] P1 pass rate ≥ 95% (≥17 of 18 passing)
- [ ] All SEC tests (E11-R-003, E11-R-005) passing at 100%
- [ ] ESPD XML exports validate against EU ESPD XSD (E11-R-002 mitigation verified)
- [ ] All 5 E2E journeys (S11.16) passing in staging environment
- [ ] No open P0/P1 severity bugs
- [ ] Agent error handling verified for all 8 new E11 agent types (E11-R-001 mitigation verified)

---

## Test Coverage Plan

> **Priority clarification:** P0/P1/P2/P3 = risk-based priority, not execution timing. See Execution Strategy section for when each priority runs.

### P0 — Critical

**Criteria:** Blocks core functionality + high risk (score ≥ 6) + no workaround

| Test ID | Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|---------|-------------|------------|-----------|------------|-------|-------|
| E11-P0-001 | ESPD profile company RLS — Company A JWT cannot GET/PATCH/DELETE Company B profile | API | E11-R-005 | 3 | QA | 404 expected (not 403) to prevent enumeration |
| E11-P0-002 | ESPD Auto-Fill: timeout → structured 503 (not raw 500); retry flow for transient failure | API | E11-R-001 | 2 | QA | AI Gateway mock: inject timeout, then 1-retry success |
| E11-P0-003 | ESPD XML export: generated XML validates against EU ESPD XSD (Parts II-V all present) | API | E11-R-002 | 2 | QA | Use python `lxml` or equivalent to validate against XSD |
| E11-P0-004 | Grant Eligibility Agent: structured 503 on timeout; structured list returned on success | API | E11-R-001 | 2 | QA | Mock: success + timeout scenarios |
| E11-P0-005 | Budget Builder: line items sum to total_budget; overhead correctly applied; co-financing split sums to total_requested_funding | API | E11-R-004 | 3 | QA | Parse valid response + inject inconsistent response → validate error |
| E11-P0-006 | Compliance Framework: non-admin JWT returns 403 on POST/GET/PATCH/DELETE /admin/compliance-frameworks | API | E11-R-003 | 4 | QA | Test company JWT, expired admin JWT (401), valid admin JWT (200) |
| E11-P0-007 | Framework Suggestion admin-only: company JWT returns 403 on GET/PATCH /admin/framework-suggestions | API | E11-R-003 | 2 | QA | Covers S11.09 admin endpoints |
| E11-P0-008 | Agent error handling: all 8 E11 agent-backed endpoints return `{"message": "...", "code": "AGENT_UNAVAILABLE"}` structure on agent 503 | API | E11-R-001 | 8 | QA | One test per agent type; mock each with 503; assert error shape consistent |
| E11-P0-009 | AI Gateway mock: all 8 E11 agent types return deterministic fixture responses in CI (smoke gate) | API | E11-R-001, TB-02 | 8 | QA | Prerequisite validation; if any agent mock is missing, sprint gate blocks |
| E11-P0-010 | 30s timeout enforced on all E11 agent endpoints; request does not hang past timeout | API | E11-R-001 | 4 | QA | Mock delayed response at 31s; assert 503 within 31s |

**Total P0:** 38 assertions across 10 test functions (~20–35 hours)

### P1 — High

**Criteria:** Important features + medium-to-high risk + common workflows

| Test ID | Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|---------|-------------|------------|-----------|------------|-------|-------|
| E11-P1-001 | Grant Eligibility: full-match, partial-match, no-match scenarios; filter params applied (programme type, funding range) | API | E11-R-001 | 3 | QA | Mock 3 fixture variants |
| E11-P1-002 | Budget Builder: per-partner breakdown present when consortium_size > 1; co_financing_split visualizable | API | E11-R-004 | 2 | QA | Mock multi-partner response |
| E11-P1-003 | Consortium Finder: paginated results; capability overlap ranking; single-country filter; max_results honoured | API | — | 3 | QA | Mock ranked list |
| E11-P1-004 | Logframe Generator: all 4 output fields present (logical_framework, work_packages, gantt_data, deliverable_table) | API | E11-R-008 | 2 | QA | Assert complete structure |
| E11-P1-005 | Logframe: gantt_data absent in response → partial result with null gantt_data returned (no 500) | API | E11-R-008 | 1 | QA | Parser graceful degradation |
| E11-P1-006 | Reporting Template Generator: project data loaded from DB, agent called, pre-filled report returned as JSON | API | — | 2 | QA | Use TB-01 seeded project |
| E11-P1-007 | Reporting Template: DOCX export generated, Content-Type `application/vnd.openxmlformats-officedocument.wordprocessingml.document`, download succeeds | API | — | 1 | QA | Use `python-docx` output; validate MIME type + non-empty body |
| E11-P1-008 | ESPD CRUD: create, list, get, update, delete — all 5 endpoints functional for company user | API | — | 5 | QA | Standard CRUD smoke |
| E11-P1-009 | ESPD espd_data structure validation: missing Part III (exclusion grounds) returns 422 with field detail | API | E11-R-002 | 2 | QA | Validate schema enforcement at API layer |
| E11-P1-010 | Compliance Framework CRUD: create, list with filters (country, regulation_type, is_active), get, update, soft-delete | API | — | 5 | QA | Admin JWT for all |
| E11-P1-011 | Framework assignment: assign 1 framework to opportunity, list, remove — CRUD complete | API | E11-R-010 | 3 | QA | |
| E11-P1-012 | Hybrid assignment: assign 2 frameworks (national + EU) to same opportunity; list returns both | API | E11-R-010 | 2 | QA | |
| E11-P1-013 | Framework auto-suggestion: Framework Suggestion Agent called on opportunity ingest; suggestions stored with confidence scores in queue | API | E11-R-007 | 2 | QA | Mock agent with 2 suggestions |
| E11-P1-014 | Suggestion accept flow: PATCH suggestion → status=accepted + opportunity framework assignment created atomically | API | E11-R-007 | 2 | QA | Assert both DB rows created |
| E11-P1-015 | Suggestion reject flow: PATCH suggestion status=rejected → no auto-assignment created | API | E11-R-007 | 1 | QA | |
| E11-P1-016 | Regulation Tracker Celery task: fires with mocked agent, regulatory_changes records created with correct fields | API | E11-R-006 | 2 | QA | Trigger manually via test helper |
| E11-P1-017 | Regulation Tracker: acknowledge + dismiss flows; acknowledged change links to affected framework | API | E11-R-006 | 2 | QA | |
| E11-P1-018 | Framework deletion guard: cannot DELETE framework assigned to active opportunity → 409 Conflict | API | E11-R-009 | 2 | QA | Covers hard-delete + soft-delete path |

**Total P1:** 42 assertions across 18 test functions (~20–35 hours)

### P2 — Medium

**Criteria:** Secondary features + low-to-medium risk + edge cases

| Test ID | Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|---------|-------------|------------|-----------|------------|-------|-------|
| E11-P2-001 | Budget Builder: missing optional params (overhead_rate absent) → defaults applied or clear validation error | API | — | 1 | QA | |
| E11-P2-002 | Consortium Finder: empty results set (no matches); contact_info absent in partner → null field, not crash | API | E11-R-012 | 2 | QA | |
| E11-P2-003 | Logframe: DOCX reporting export — `POST /grants/reporting-template/export` returns valid DOCX | API | — | 1 | QA | |
| E11-P2-004 | ESPD CRUD cross-company: Company A cannot PATCH Company B profile (404 expected) | API | E11-R-005 | 2 | QA | Covers PATCH + DELETE |
| E11-P2-005 | ESPD espd_data: all 4 Parts can be independently patched without overwriting other Parts | API | — | 1 | QA | Partial PATCH semantics |
| E11-P2-006 | Compliance Framework rules JSONB: invalid rule schema (missing `criterion` field) → 422 | API | — | 1 | QA | |
| E11-P2-007 | Framework suggestion: override with alternative framework_id → override framework assigned | API | E11-R-007 | 1 | QA | |
| E11-P2-008 | Regulation tracker: GET /admin/regulatory-changes filters (status, severity, date_range) all functional | API | — | 3 | QA | |
| E11-P2-009 | Platform settings: GET /admin/platform-settings (admin only); PATCH /admin/platform-settings/:key merges value | API | — | 2 | QA | |
| E11-P2-010 | Platform settings: invalid key returns 404; PATCH with invalid JSON value returns 422 | API | — | 2 | QA | |
| E11-P2-011 | Grant Eligibility Panel: loading spinner during agent call; error state on 503; empty state if no matches | E2E | — | 3 | QA | Playwright; mock via route() |
| E11-P2-012 | Budget Builder Panel: editable table cells recalculate totals on input change | E2E | E11-R-004 | 2 | QA | Frontend arithmetic validation |
| E11-P2-013 | Consortium Finder Panel: tag input + multi-select country render; partner card grid displays all fields | E2E | — | 2 | QA | |
| E11-P2-014 | Logframe Panel: Gantt chart renders with tasks; deliverable table is sortable | E2E | — | 2 | QA | |
| E11-P2-015 | ESPD Profile List: empty state shown; "Create New Profile" navigates to editor | E2E | — | 2 | QA | |
| E11-P2-016 | ESPD Profile Editor: multi-step form with inline validation; Part III exclusion grounds checkboxes | E2E | — | 2 | QA | |
| E11-P2-017 | ESPD Auto-Fill Preview: side-by-side diff renders; changed fields highlighted; download XML button functional | E2E | — | 2 | QA | Mock auto-fill via route() |
| E11-P2-018 | Compliance Framework List: filter by country + regulation_type; search by name; pagination | E2E | — | 2 | QA | |
| E11-P2-019 | Framework Assignment Page: add 2 frameworks to opportunity; remove 1; list shows remaining | E2E | E11-R-010 | 2 | QA | |
| E11-P2-020 | Auto-Suggestion Queue: batch accept; confidence bar colour-coded; filter by confidence threshold | E2E | E11-R-007 | 2 | QA | |

**Total P2:** 37 assertions across 20 test functions (~15–25 hours)

### P3 — Low

**Criteria:** Critical user journeys (E2E integration from S11.16) + performance benchmarks

| Test ID | Requirement | Test Level | Test Count | Owner | Notes |
|---------|-------------|------------|------------|-------|-------|
| E11-P3-001 | E2E Journey 1 (S11.16): company runs eligibility check → views matched programmes → triggers budget builder → views budget | E2E | 1 | QA | Staging; mock agents |
| E11-P3-002 | E2E Journey 2 (S11.16): company creates ESPD profile → triggers auto-fill → previews result → exports as XML | E2E | 1 | QA | Staging; validate XML download |
| E11-P3-003 | E2E Journey 3 (S11.16): admin creates compliance framework → assigns to opportunity → user's proposal validated against it (reuses E07 compliance checker) | E2E | 1 | QA | Requires E07 stable |
| E11-P3-004 | E2E Journey 4 (S11.16): new opportunity ingested → framework suggestion generated → admin accepts → framework assigned automatically | E2E | 1 | QA | |
| E11-P3-005 | E2E Journey 5 (S11.16): regulation tracker fires → admin views change → acknowledges → reviews affected framework | E2E | 1 | QA | Manual Celery trigger |
| E11-P3-006 | Reporting Template DOCX: downloaded file is valid Word document (opens without error; contains structured sections) | API | 1 | QA | python-docx verify |
| E11-P3-007 | Regulation Tracker Frontend: feed renders change cards; acknowledge + dismiss buttons functional; acknowledged change links to framework | E2E | 1 | QA | |
| E11-P3-008 | k6: 10 concurrent agent endpoint calls under 30s timeout constraint; verify p95 < 5s on mock; no 500s | Perf | 1 | QA | k6 nightly; AI Gateway mock only |

**Total P3:** 8 test functions (~10–20 hours)

---

## Execution Strategy

**Philosophy:** Run everything in PRs if the suite completes in under 15 minutes. Playwright parallelizes 100+ tests in 10–15 min. Only defer expensive or long-running tests.

**Every PR:**
- All P0 + P1 + P2 API tests (pytest + pytest-asyncio): ~10–12 min
- P2 E2E tests (Playwright, parallelized, AI Gateway mocked via `route()`): ~5–8 min

**Nightly:**
- P3 E2E journeys (full stack integration, staging environment): ~15–25 min
- k6 agent load test (P3-008): ~10 min

**Weekly:**
- ESPD XML conformance validation against latest EU ESPD XSD (E11-R-002 regression)
- DOCX format validation (P3-006, P3-007 visual check)
- Regulation Tracker manual E2E journey with real Celery Beat trigger

---

## Resource Estimates

| Priority | Count | Effort Range | Notes |
|----------|-------|-------------|-------|
| P0 | 10 | ~20–35 hours | AI Gateway mock setup for 8 agent types; ESPD XSD integration; complex auth scenarios |
| P1 | 18 | ~20–35 hours | Standard API CRUD + Celery task triggers; fixture data for agent types |
| P2 | 20 | ~15–25 hours | Mix of API edge cases and Playwright E2E for frontend panels |
| P3 | 8 | ~10–20 hours | E2E journeys against staging; k6 agent load script |
| **Total** | **56** | **~65–115 hours** | **~2–3.5 weeks (1 QA)**; parallelisable between backend and frontend QA |

### Prerequisites

**Test Data (TB-01 extensions required):**
- `espd_profile` factory: company_id FK, profile_name, espd_data with all 4 Parts
- `compliance_framework` factory: name, country, regulation_type, rules JSONB
- `opportunity_compliance_frameworks` factory: opportunity_id + framework_id join records
- `regulatory_change` factory: source, change_type, severity, detected_at, affected_frameworks
- `framework_suggestion` factory: opportunity_id, framework_id, confidence, status=pending

**AI Gateway Mock (TB-02 extensions required):**

| Agent Type | Required Fixture Responses |
|-----------|---------------------------|
| Grant Eligibility Agent | Full-match (3 programmes), partial-match (1 programme), no-match |
| Budget Builder Agent | Valid budget (consistent arithmetic), inconsistent arithmetic (for error path test) |
| Consortium Finder Agent | Ranked list (3 partners), empty results, partner with missing contact_info |
| Logframe Generator Agent | Complete structure, gantt_data absent (partial) |
| Reporting Template Generator Agent | Pre-filled periodic report JSON |
| ESPD Auto-Fill Agent | Pre-filled ESPD data (changed fields highlighted) |
| Framework Suggestion Agent | 2 suggestions with confidence scores |
| Regulation Tracker Agent | 1 new regulatory change record |

**All 8 agent types must also support:** configurable 503 failure injection, configurable 30s+ delayed response (for timeout tests)

**Tooling:**
- `pytest` + `pytest-asyncio` + `httpx` — backend API tests
- `Playwright` + `playwright-utils` (API-only + UI profiles) — E2E + frontend panel tests
- `lxml` (Python) — EU ESPD XSD validation for XML conformance tests
- `k6` — agent endpoint load test (P3-008)
- `python-docx` — DOCX structural validation

**Environment:**
- Dev: all P0 + P1 + P2 API tests (mocked AI Gateway)
- Staging: P3 E2E journeys (requires KraftData agents pre-configured)
- CI: PR suite (P0 + P1 + P2) must complete < 15 min

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate:** 100% (no exceptions; blocks merge)
- **P1 pass rate:** ≥ 95% (≥17 of 18; failures require triage ticket before merge)
- **P2/P3 pass rate:** ≥ 90% (informational; failures logged as tech debt)

### Non-Negotiable Requirements

- [ ] All P0 tests pass
- [ ] SEC tests 100%: E11-P0-001 (ESPD RLS), E11-P0-006 (admin 403), E11-P0-007 (suggestions 403)
- [ ] ESPD XML exports validate against EU ESPD XSD (E11-P0-003 + E11-P1-009)
- [ ] Agent error handling consistent across all 8 E11 agent types (E11-P0-008 + E11-P0-010)
- [ ] No open P0/P1 severity bugs at sprint exit
- [ ] All 5 S11.16 E2E journeys passing in staging before MVP release

---

## Mitigation Plans

### E11-R-001: AI Gateway Error Handling — 8 New Agent Types (Score: 6)

**Mitigation Strategy:**
1. Extend TB-02 mock mode with fixture responses for all 8 E11 agent types — done before Sprint 11 testing begins
2. Add configurable failure injection per agent type: `"mode": "timeout"` (31s delay), `"mode": "503"`, `"mode": "transient"` (503 then success on retry)
3. For every E11 agent-backed endpoint: test with forced timeout → assert structured 503 with `{"message": "AI features temporarily unavailable", "code": "AGENT_UNAVAILABLE"}` — not a raw 500
4. Test retry logic: inject 1 transient 503 on first call → verify second call succeeds → verify response served to user
5. Test graceful degradation UI: Playwright asserts "temporarily unavailable" message (not loading spinner) after mock 503

**Owner:** Backend Lead (mock extension) + QA (test implementation) | **Timeline:** Sprint 11 | **Status:** Planned  
**Verification:** E11-P0-002, E11-P0-004, E11-P0-008, E11-P0-010 all green; each of 8 agent types covered

---

### E11-R-002: ESPD XML Schema Conformance (Score: 6)

**Mitigation Strategy:**
1. Obtain official EU ESPD XSD schema v2.1+ from https://github.com/ESPD/ESPD-EDM or EU eProcurement gateway
2. Add `lxml` XML schema validator to test suite; load XSD once as fixture
3. POST /espd-profiles/:id/export?format=xml → capture XML response → validate against XSD; assert no validation errors
4. Validate both paths: (a) manually completed ESPD profile, (b) auto-filled profile via ESPD Auto-Fill Agent
5. Validate all required Parts present: Part II (economic operator info), Part III (exclusion grounds), Part IV (selection criteria), Part V (reduction of candidates)

**Owner:** Backend Lead | **Timeline:** Sprint 11 | **Status:** Planned  
**Verification:** E11-P0-003 (XSD validation passes) + E11-P1-009 (schema enforcement at API layer)

---

### E11-R-003: Admin Endpoint Authorization (Score: 6)

**Mitigation Strategy:**
1. For every S11.08/09/10 endpoint: test with company JWT → assert 403 Forbidden
2. Test with expired admin JWT → assert 401 Unauthorized
3. Test with valid admin JWT → assert 2xx success
4. Verify admin routes are not accessible via company API path (different base path `/admin/` vs `/`)
5. Verify admin-only middleware applied to ALL S11.10 routes (platform-settings, regulatory-changes)

**Owner:** Backend | **Timeline:** Sprint 11 | **Status:** Planned  
**Verification:** E11-P0-006, E11-P0-007 pass; all 3 endpoint groups covered (frameworks, assignments, regulations)

---

### E11-R-004: Budget Arithmetic Consistency (Score: 6)

**Mitigation Strategy:**
1. Test with valid AI response where sum(cost_categories.amount) === totals.total_direct_costs — assert parser serves this correctly
2. Inject AI response where line items don't sum to total → assert backend returns 422 (not 200 with bad data)
3. Test overhead: `indirect_costs = overhead_rate * total_direct_costs` — assert calculation is exact (not approximate)
4. Test co-financing split: `eu_contribution + own_contribution === total_requested_funding` — assert exact match
5. Test per-partner breakdown: sum(partner_amounts) === total_requested_funding — assert for consortium_size ≥ 2

**Owner:** Backend Lead | **Timeline:** Sprint 11 | **Status:** Planned  
**Verification:** E11-P0-005, E11-P1-002 pass; inconsistent AI response rejected with 422, not silently served

---

### E11-R-005: ESPD Company-Scoped RLS (Score: 6)

**Mitigation Strategy:**
1. Seed: Company A user + 1 ESPD profile; Company B user + 1 ESPD profile (different UUIDs)
2. Company A JWT → GET /espd-profiles/{company_B_profile_id} → assert 404 (not 403, to prevent enumeration)
3. Company A JWT → PATCH /espd-profiles/{company_B_profile_id} → assert 404
4. Company A JWT → DELETE /espd-profiles/{company_B_profile_id} → assert 404
5. Company A JWT → POST /espd-profiles with explicit `company_id: company_B.id` in body → assert company_id overridden to Company A's id (JWT-derived)
6. Company A JWT → POST /espd-profiles/:id/auto-fill for Company B's profile ID → assert 404

**Owner:** Backend | **Timeline:** Sprint 11 | **Status:** Planned  
**Verification:** E11-P0-001 (3 test cases) + E11-P2-004 pass

---

## Assumptions and Dependencies

### Assumptions

1. The EU ESPD XSD v2.1+ schema is publicly accessible (https://github.com/ESPD/ESPD-EDM) and stable enough to pin for test validation
2. All 8 E11 KraftData agent types (Grant Eligibility, Budget Builder, Consortium Finder, Logframe Generator, Reporting Template Generator, ESPD Auto-Fill, Framework Suggestion, Regulation Tracker) are pre-configured in KraftData before staging E2E testing begins
3. S11.01 Alembic migrations run successfully; dev DB seeded with sample compliance frameworks and ESPD profiles before Sprint 11 testing
4. TB-02 mock mode is extended by Backend Lead before Sprint 11 testing begins; all 8 E11 agent types return fixture responses with failure injection support
5. Celery Beat can be triggered in test environments via a manual trigger endpoint or test utility (not requiring wall-clock wait)
6. E04 (AI Gateway) 30s timeout and 1-retry-with-backoff are configurable per agent type, not just globally

### Dependencies

| Dependency | Owner | Required By | Reason |
|-----------|-------|------------|--------|
| TB-01: test data seeding extended (ESPD, frameworks, suggestions, regulatory changes) | Backend Lead | Sprint 11 start | All integration + E2E tests |
| TB-02: AI Gateway mock — 8 new E11 agent types, failure injection | Backend Lead | Sprint 11 start | All P0 agent tests blocked without mock |
| EU ESPD XSD v2.1+ obtained and added to test fixtures | Backend Lead / QA | Sprint 11 | E11-R-002 mitigation |
| Celery Beat test trigger utility | Backend Lead | Sprint 11 | S11.10 Regulation Tracker tests |
| E04 (AI Gateway) stable with configurable timeouts per agent type | Backend Lead | Sprint 11 start | E11-R-001 mitigation |
| E06 (billing) + E07 (compliance checker) stable | Dev Team | Sprint 11 start | E11 dependency per epic header; E2E journey 3 requires E07 |

### Risks to Plan

- **Risk:** TB-02 mock not extended with all 8 E11 agent types before Sprint 11  
  **Impact:** E11-P0-008 and E11-P0-009 gates fail; ~40% of P0 tests blocked  
  **Contingency:** Playwright `route()` interceptor at HTTP level to stub KraftData responses for individual agent endpoints as TB-02 stopgap

- **Risk:** EU ESPD XSD not available or changes mid-sprint  
  **Impact:** E11-P0-003 and E11-R-002 mitigation blocked; ESPD XML validation manual only  
  **Contingency:** Use representative ESPD XML sample from EU eProcurement documentation as hand-crafted validation fixture; pin to specific ESPD-EDM commit hash

- **Risk:** E07 (compliance checker) not stable before E2E journey 3 (E11-P3-003)  
  **Impact:** S11.16 journey 3 blocked; defer to post-E07 stabilisation  
  **Contingency:** Mock compliance checker response in E11-P3-003 using `route()` until E07 is stable; flag as partial test

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
|-------------------|--------|------------------|
| **E04 — AI Gateway** | All 8 E11 agent types route through AI Gateway; circuit breaker, timeout, and retry logic must handle new agent types | Re-run E04 P0 smoke (health, circuit breaker open/close) before E11 agent tests |
| **E07 — Compliance Checker** | S11.16 E2E journey 3 directly invokes E07 compliance checker against an E11-managed framework | E07 P0 tests must pass; E11-P3-003 depends on E07 stability |
| **E02 — Auth/RBAC** | S11.08–S11.10 admin-only middleware reuses E02 JWT validation and RBAC infrastructure | E02 P0-001 (JWT validation) and P0-005 (RBAC ceiling) must remain green |
| **Data Pipeline (E04 crawlers)** | Framework Suggestion Agent triggered on opportunity ingestion event from pipeline | E04 pipeline ingestion event format must remain compatible with S11.09 suggestion trigger hook |
| **ESPD Profile (E02 S02.12)** | E02 included an ESPD Profile CRUD story (S02.12); E11 S11.01–S11.03 extends this schema and adds auto-fill | Verify S02.12 tests still pass after S11.01 schema migration; no breaking changes to espd_profiles table structure |

---

## Follow-on Workflows

- Run `*atdd` to generate failing P0 acceptance tests from E11-P0 scenarios (separate workflow; not auto-run).
- Run `*automate` for broader coverage expansion once S11.01–S11.10 implementation is in place.

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk classification framework (P×I scoring, gate decision rules)
- `probability-impact.md` — Probability/impact scale definitions (1=unlikely/minor, 3=likely/critical)
- `test-levels-framework.md` — Test level selection (E2E only for critical paths; API for business logic)
- `test-priorities-matrix.md` — P0–P3 prioritization criteria

### Related Documents

- **Epic:** `eusolicit-docs/planning-artifacts/epic-11-grants-compliance.md`
- **System-Level Architecture:** `eusolicit-docs/test-artifacts/test-design-architecture.md` (v3, 2026-04-09)
- **System-Level QA:** `eusolicit-docs/test-artifacts/test-design-qa.md` (v5, 2026-04-09)
- **Handoff:** `eusolicit-docs/test-artifacts/test-design/eu-solicit-handoff.md`
- **Prior Epic Designs:** test-design-epic-01.md, test-design-epic-02.md, test-design-epic-03.md, test-design-epic-12.md

---

**Generated by:** BMad TEA Agent — Test Architect Module  
**Workflow:** `bmad-testarch-test-design`  
**Version:** 4.0 (BMad v6)
