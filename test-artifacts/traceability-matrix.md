---
stepsCompleted:
  - step-01-load-context
  - step-02-discover-tests
  - step-03-map-criteria
  - step-04-analyze-gaps
  - step-05-gate-decision
lastStep: step-05-gate-decision
lastSaved: '2026-04-11'
epic: 11
epicTitle: 'EU Grant Specialization & Compliance'
workflowType: bmad-testarch-trace
inputDocuments:
  - eusolicit-docs/planning-artifacts/epic-11-grants-compliance.md
  - eusolicit-docs/test-artifacts/test-design-epic-11.md
  - eusolicit-docs/test-artifacts/atdd-checklist-11-1-espd-profile-compliance-framework-db-schema-migrations.md
  - eusolicit-docs/test-artifacts/atdd-checklist-11-2-espd-profile-crud-api.md
  - eusolicit-docs/test-artifacts/atdd-checklist-11-3-espd-auto-fill-agent-integration.md
  - eusolicit-docs/test-artifacts/atdd-checklist-11-4-grant-eligibility-agent-integration.md
  - eusolicit-docs/test-artifacts/atdd-checklist-11-5-budget-builder-agent-integration.md
  - eusolicit-docs/test-artifacts/atdd-checklist-11-6-consortium-finder-agent-integration.md
  - eusolicit-docs/test-artifacts/atdd-checklist-11-7-logframe-generator-reporting-template-agent-integrations.md
tddPhase: RED (S11.01–S11.07 only)
---

# Traceability Matrix & Quality Gate Report

**Epic:** E11 — EU Grant Specialization & Compliance
**Generated:** 2026-04-11
**Scope:** 16 stories (S11.01–S11.16), 56 epic test IDs (P0: 10 · P1: 18 · P2: 20 · P3: 8)
**TDD Phase:** 🔴 RED — S11.01–S11.07: 210 tests written (all skipped/RED); S11.08–S11.16: no tests written yet
**Sprint:** 11–12 | **Points:** 55 | **Dependencies:** E04, E06, E07 | **Milestone:** MVP

---

## TRACE_GATE: FAIL

**Rationale:** P0 coverage is 40% FULL (required: 100%). Two P0 requirements — E11-P0-006
(Compliance Framework admin auth) and E11-P0-007 (Framework Suggestion admin auth) — have **zero**
test coverage. Three additional P0 requirements (E11-P0-008, E11-P0-009, E11-P0-010) cover only
6 of 8 required agent types (Framework Suggestion Agent and Regulation Tracker Agent have no tests).
P1 FULL coverage is 44.4% (required: ≥80% minimum). Nine of 18 P1 test IDs (all admin-side
scenarios from S11.08–S11.10) are completely uncovered. Overall FULL coverage is 30.4% (minimum:
80%). Stories S11.08–S11.16 (9 stories = backend admin, all 5 frontend stories, and E2E story)
have no ATDD checklists.

> **📋 TDD Baseline Context:** All 210 existing tests (S11.01–S11.07) are in 🔴 RED phase —
> tests are written but implementations not yet started. The gate measures traceability coverage
> (whether every acceptance criterion maps to at least one test), not execution pass/fail.
> Stories S11.08–S11.16 need ATDD checklists written before this gate can improve.

> **✅ To reach PASS:** Run `bmad-testarch-atdd` for S11.08 (Compliance Framework CRUD Admin),
> S11.09 (Framework Assignment & Auto-Suggestion), S11.10 (Regulation Tracker & Platform Settings),
> S11.11–S11.15 (all frontend stories), and S11.16 (E2E Integration). Then re-run this gate.

---

## 1. Coverage Statistics

| Dimension | FULL | PARTIAL | NONE | Total | % FULL | % Covered |
|-----------|-----:|--------:|-----:|------:|-------:|----------:|
| **P0**    |  4   |    4    |  2   |  10   | **40%**  | 80%     |
| **P1**    |  8   |    1    |  9   |  18   | **44.4%**| 50%     |
| **P2**    |  5   |    0    | 15   |  20   | **25%**  | 25%     |
| **P3**    |  0   |    1    |  7   |   8   | **0%**   | 12.5%   |
| **Overall** | **17** | **6** | **33** | **56** | **30.4%** | **41.1%** |

### Gate Criteria Evaluation

| Criterion | Required | Actual | Status |
|-----------|----------|--------|--------|
| P0 FULL coverage | 100% | 40% | ❌ NOT MET — 2 P0 tests with zero coverage; 4 P0 partially covered |
| P1 FULL coverage (PASS target) | ≥ 90% | 44.4% | ❌ NOT MET |
| P1 FULL coverage (minimum) | ≥ 80% | 44.4% | ❌ NOT MET |
| Overall FULL coverage | ≥ 80% | 30.4% | ❌ NOT MET |

### Test Volume by Story

| Story | Title | Tests Written | TDD Phase | Test File(s) |
|-------|-------|-------------:|-----------|--------------|
| S11.01 | ESPD Profile & Compliance Framework DB Schema + Migrations | 39 | 🔴 RED | `test_009_migration.py`, `admin-api/test_002_migration.py` |
| S11.02 | ESPD Profile CRUD API | 41 | 🔴 RED | `test_espd_profile.py` |
| S11.03 | ESPD Auto-Fill Agent Integration | 27 | 🔴 RED | `test_espd_autofill_export.py` |
| S11.04 | Grant Eligibility Agent Integration | 15 | 🔴 RED | `test_grant_eligibility.py` |
| S11.05 | Budget Builder Agent Integration | 27 | 🔴 RED | `test_budget_builder.py` |
| S11.06 | Consortium Finder Agent Integration | 26 | 🔴 RED | `test_consortium_finder.py` |
| S11.07 | Logframe Generator & Reporting Template | 35 | 🔴 RED | `test_logframe_generator.py`, `test_reporting_template.py` |
| S11.08 | Compliance Framework CRUD API (Admin) | **0** | ⚫ NO TESTS | — |
| S11.09 | Framework Assignment & Auto-Suggestion API (Admin) | **0** | ⚫ NO TESTS | — |
| S11.10 | Regulation Tracker Agent & Platform Settings API (Admin) | **0** | ⚫ NO TESTS | — |
| S11.11 | EU Grant Tools Frontend — Eligibility & Budget Panels | **0** | ⚫ NO TESTS | — |
| S11.12 | EU Grant Tools Frontend — Consortium Finder & Logframe Panels | **0** | ⚫ NO TESTS | — |
| S11.13 | ESPD Profile Management & Auto-Fill Frontend | **0** | ⚫ NO TESTS | — |
| S11.14 | Compliance Admin — Framework Management Frontend | **0** | ⚫ NO TESTS | — |
| S11.15 | Compliance Admin — Assignment, Suggestions & Regulation Tracker Frontend | **0** | ⚫ NO TESTS | — |
| S11.16 | E2E Integration Testing & Agent Error Handling Hardening | **0** | ⚫ NO TESTS | — |
| **Total** | | **210** | **S11.01–07 RED; S11.08–16 none** | 9 test files |

---

## 2. Epic-Level Acceptance Criteria Traceability

*The 13 Epic-level acceptance criteria from `epic-11-grants-compliance.md`.*

| # | Epic AC | Tests | Level | Priority | Coverage |
|---|---------|-------|-------|----------|----------|
| AC-E1 | Grant Eligibility Agent maps company profile → matched programmes with eligibility scores | S11.04: `TestAC1AC2HappyPath` (5 tests), `TestAC3ResponseParsing` (4 tests), `TestAC4AgentErrorHandling` (3 tests) | API | P0/P1 | **FULL** |
| AC-E2 | Budget Builder Agent generates EU-compliant budget with cost categories, overhead, co-financing | S11.05: `TestAC1AC2HappyPath` (4 tests), `TestAC3ResponseParsing` (4 tests), `TestAC4ArithmeticValidation` (4 tests), `TestAC5ConsortiumBudget` (4 tests) | API | P0/P1 | **FULL** |
| AC-E3 | Consortium Finder Agent searches Consortium Partners Store and returns ranked partner suggestions | S11.06: `TestAC1AC2HappyPath` (5 tests), `TestAC3AC5ResponseParsing` (6 tests), `TestAC6GracefulDegradation` (4 tests) | API | P1 | **FULL** |
| AC-E4 | Logframe Generator Agent produces logical frameworks, work packages, Gantt chart data, deliverables | S11.07 logframe: `TestAC1AC2HappyPath` (4 tests), `TestAC3AC4ResponseParsing` (7 tests) | API | P1 | **FULL** |
| AC-E5 | Reporting Template Generator Agent pre-fills periodic report templates from awarded project data | S11.07 reporting: `TestAC7AC8HappyPath` (5 tests), `TestAC11DOCX` (4 tests) | API | P1 | **FULL** |
| AC-E6 | ESPD profiles can be created, edited, listed, and deleted with structured data mapped to ESPD XML schema (Parts II-V) | S11.01: schema migration tests (E11-DB-001 to E11-DB-009); S11.02: all CRUD tests (AC1–AC5, AC9) | DB + API | P1 | **FULL** |
| AC-E7 | ESPD Auto-Fill Agent maps company profile data to ESPD fields → pre-filled ESPD downloadable as XML and PDF | S11.03: `TestAC1AutoFillHappyPath` (4), `TestAC5ExportXml` (5), `TestAC7ExportPdf` (3) | API | P0 | **PARTIAL** *(XSD full validation weekly only; structural tests present)* |
| AC-E8 | Admins can create, edit, activate/deactivate, and delete compliance frameworks with structured validation rules | **No ATDD checklist for S11.08** | — | P0/P1 | **NONE** |
| AC-E9 | Admins can assign one or more compliance frameworks per opportunity, supporting hybrid national+EU scenarios | **No ATDD checklist for S11.09** | — | P1 | **NONE** |
| AC-E10 | Framework Suggestion Agent auto-suggests applicable frameworks; admins confirm or override | **No ATDD checklist for S11.09** | — | P0/P1 | **NONE** |
| AC-E11 | Regulation Tracker Agent runs on Celery Beat schedule, surfaces regulatory changes; admins acknowledge/dismiss | **No ATDD checklist for S11.10** | — | P1 | **NONE** |
| AC-E12 | All agent calls go through AI Gateway with proper error handling, loading states, and timeout management | S11.03–S11.07: error handling for ESPD Auto-Fill, Grant Eligibility, Budget Builder, Consortium Finder, Logframe Generator, Reporting Template (6 agents). Framework Suggestion + Regulation Tracker: **NOT COVERED** | API | P0 | **PARTIAL** *(6/8 agents)* |
| AC-E13 | Frontend provides dedicated pages for grant tools, ESPD management, and compliance administration | **No ATDD checklists for S11.11–S11.15** | — | P2 | **NONE** |

---

## 3. Story-Level Acceptance Criteria Traceability

### S11.01 — ESPD Profile & Compliance Framework DB Schema + Migrations

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Migration 009 runs from 008 baseline; espd_profiles restructured with profile_name + espd_data | E11-DB-001 → E11-DB-009 (client-api): `TestE11DB001MigrationLifecycle` (3), `TestE11DB007DataMigration` (2), `TestE11DB009Downgrade` (1), `TestE11DB002TableColumnStructure` (4), `TestE11DB003UniqueConstraintRemoval` (1), `TestE11DB004And005ColumnConstraints` (2), `TestE11DB006ServerDefaultBehavior` (1) | Integration | P0/P1 | **FULL** |
| AC2 | Data migration: field_values → espd_data['part_iii'], profile_name = 'Migrated Profile' | `TestE11DB007DataMigration`: `test_field_values_migrated_to_espd_data_part_iii`, `test_empty_field_values_row_gets_empty_espd_data` | Integration | P0 | **FULL** |
| AC3 | Admin-api migration 002: creates compliance_frameworks, platform_settings, opportunity_compliance_frameworks | E11-DB-010 → E11-DB-018 (admin-api): all 10 test classes, 22 test functions | Integration | P0 | **FULL** |
| AC4 | All required indexes present in pg_indexes; ix_espd_profiles_company_id retained | E11-DB-008 (client-api), E11-DB-017 (admin-api): 8 tests | Integration | P0 | **FULL** |
| AC5 | ORM model files created in admin-api | `test_orm_models_directory_created` (E11-DB-010 class) | Integration | P1 | **FULL** |
| AC6–7 | *(implicit in AC1–AC3: column constraints, NOT NULL, defaults, FK cascade)* | E11-DB-004, E11-DB-005, E11-DB-016 | Integration | P0 | **FULL** |
| AC8 | `alembic check` shows no pending changes (ORM ↔ DB sync) | `test_alembic_check_shows_no_pending_changes` (both services) | Integration | P1 | **FULL** |

**Story Coverage:** 8/8 ACs → **FULL** | 39 tests total (P0: 26, P1: 13)

---

### S11.02 — ESPD Profile CRUD API

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `POST /espd-profiles` → HTTP 201; company_id from JWT; admin/bid_manager only; others → 403 | `TestAC1CreateEspdProfile` (7 tests) + `TestAC9RoleEnforcement` (post variants) | API | P1/P0 | **FULL** |
| AC2 | `GET /espd-profiles` → HTTP 200; all company profiles ordered by created_at DESC | `TestAC2ListEspdProfiles` (4 tests) | API | P1 | **FULL** |
| AC3 | `GET /espd-profiles/{id}` → 200 (own) or 404 (cross-company); no 403 | `TestAC3GetEspdProfile` (3 tests) | API | P1 | **FULL** |
| AC4 | `PATCH /espd-profiles/{id}` → 200; profile_name replaced; espd_data merged at Part level; 404 if cross-company | `TestAC4PatchEspdProfile` (6 tests) | API | P1 | **FULL** |
| AC5 | `DELETE /espd-profiles/{id}` → 204; 404 if cross-company | `TestAC5DeleteEspdProfile` (3 tests) | API | P1 | **FULL** |
| AC6 | Company-scoped RLS: all ops derive company_id from JWT; cross-company → 404 (not 403) | `TestAC6CrossCompanyRLS` (5 tests): GET/PATCH/DELETE cross-company → 404 | API | P0 | **FULL** |
| AC7 | espd_data PATCH merge: Part-level (part_ii–part_v); absent Parts left unchanged | `test_patch_espd_data_part_level_merge` | API | P2 | **FULL** |
| AC8 | espd_data validation: dict required; each Part key must be dict or absent; non-dict → 422 | `TestAC1CreateEspdProfile`: `test_post_invalid_part_type_returns_422` | API | P1 | **FULL** |
| AC9 | Unauthenticated → 401; low-privilege roles → 403 on write; GET accessible to all | `TestAC9RoleEnforcement` (5 methods, 13 parametrized cases) | API | P0 | **FULL** |

**Story Coverage:** 9/9 ACs → **FULL** | 41 tests total (P0 security: 3 RLS + auth tests)

---

### S11.03 — ESPD Auto-Fill Agent Integration

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `POST /espd-profiles/{id}/auto-fill` → 200; snapshot_profile_id, source_profile_id, espd_data, changed_fields | `TestAC1AutoFillHappyPath` (4 tests) | API | P0 | **FULL** |
| AC2 | Snapshot profile created with name "Auto-fill snapshot: …"; original profile unchanged | `test_autofill_creates_new_snapshot_profile_in_db` (2 tests) | API | P1 | **FULL** |
| AC3 | Gateway payload includes espd_data, opportunity_id, company_id; X-Caller-Service header | `TestAC3GatewayPayload` (2 tests) | API | P1 | **FULL** |
| AC4 | Gateway timeout / 5xx → 503 AGENT_UNAVAILABLE; no snapshot on failure | `TestAC4AgentErrorHandling` (4 tests) | API | P0 | **FULL** |
| AC5 | `POST /espd-profiles/{id}/export` validates format=xml\|pdf; missing/invalid → 422 | `TestExportFormatValidation` (2 tests) | API | P1 | **FULL** |
| AC6 | XML export: `<ESPDResponse xmlns="urn:X-eusolicit:espd:schema:v1">`; Parts mapped; well-formed | `TestAC5ExportXml` (5 tests): well-formed, root namespace, all Parts present | API | P0 | **PARTIAL** *(structural tests pass; full XSD lxml conformance deferred to weekly job per test-design exit criteria)* |
| AC7 | PDF export: non-empty body starting with %PDF-; correct Content-Disposition | `TestAC7ExportPdf` (3 tests) | API | P1 | **FULL** |
| AC8 | Company-scoped RLS on both endpoints; cross-company → 404 | `TestAC8CrossCompanyRLS` (2 tests) | API | P0 | **FULL** |
| AC9 | Unauthenticated → 401; contributor/reviewer/read_only → 403 | `TestAC9Authorization` (4 methods, 8 with parametrize) | API | P0 | **FULL** |
| AC10 | Empty espd_data ({}) → 422 with "no data to auto-fill" | `TestAC10EmptyEspdValidation` (1 test) | API | P1 | **FULL** |

**Story Coverage:** 9/10 ACs FULL, 1/10 PARTIAL (AC6 XSD conformance) | 27 tests total

---

### S11.04 — Grant Eligibility Agent Integration

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `POST /grants/eligibility-check` accepts optional filters; loads company; returns 200 | `test_eligibility_check_returns_200_with_response_structure` | API | P0 | **FULL** |
| AC2 | Agent payload: company_profile + filters + X-Caller-Service + 30s timeout; no client retry | `test_eligibility_check_agent_payload_*` (4 tests), `test_eligibility_check_x_caller_service_header_sent` | API | P0 | **FULL** |
| AC3 | Response parsed into GrantEligibilityResponse (matched programmes with eligibility_score, programme_name, call_reference, requirements_summary, gap_analysis) | `TestAC3ResponseParsing` (4 tests): full-match, partial-match, no-match, missing key default | API | P1 | **FULL** |
| AC4 | Gateway timeout/5xx → 503 AGENT_UNAVAILABLE; no raw error forwarded | `TestAC4AgentErrorHandling` (3 tests): timeout, HTTP 500, HTTP 503 | API | P0 | **FULL** |
| AC5 | Company not found → 404 | `test_company_not_found_returns_404` | API | P1 | **FULL** |
| AC6 | Unauthenticated → 401; all roles permitted | `TestAC6Authorization` (2 tests) | API | P0 | **FULL** |
| AC7 | No client-side filtering; null when absent; verbatim forwarding | `test_eligibility_check_with_programme_type_filter`, `test_eligibility_check_with_funding_range_filter` | API | P1 | **FULL** |

**Story Coverage:** 7/7 ACs → **FULL** | 15 tests total (P0: 9, P1: 6)

---

### S11.05 — Budget Builder Agent Integration

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `POST /grants/budget-builder` accepts JSON body; returns 200 + BudgetBuilderResponse | `TestAC1AC2HappyPath` (4 tests) + `TestAC2OptionalParams` validation tests | API | P0 | **FULL** |
| AC2 | Agent payload: all required fields + X-Caller-Service + company_id string; optional params forwarded | `test_budget_builder_agent_payload_has_required_fields`, `test_budget_builder_x_caller_service_header_sent`, `test_budget_builder_with_all_optional_params` | API | P0 | **FULL** |
| AC3 | Response parsed: cost_categories, overhead_calculation, co_financing_split, per_partner_breakdown | `TestAC3ResponseParsing` (4 tests) | API | P1 | **FULL** |
| AC4 | Budget arithmetic validation: line items, co-financing, overhead consistency | `TestAC4ArithmeticValidation` (4 tests): valid passes, 3 inconsistent → 422 | API | P0 | **FULL** |
| AC5 | Per-partner breakdown required when consortium_size > 1; null accepted when solo | `TestAC5ConsortiumBudget` (4 tests) | API | P1 | **FULL** |
| AC6 | Gateway timeout/5xx → 503 AGENT_UNAVAILABLE | `TestAC6AgentErrorHandling` (3 tests) | API | P0 | **FULL** |
| AC7 | Unauthenticated → 401; all roles permitted | `TestAC7Authorization` (2 tests) | API | P0 | **FULL** |
| AC8 | Stateless: no DB writes | Implicit via fixture teardown (no session.flush); covered architecturally | API | P2 | **FULL** |

**Story Coverage:** 8/8 ACs → **FULL** | 27 tests total (P0: 12, P1: 8, P2: 7)

---

### S11.06 — Consortium Finder Agent Integration

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `POST /grants/consortium-finder` accepts required/optional fields; returns 200 | `TestAC1AC2HappyPath` (happy path tests) + `TestAC1InputValidation` (6 tests) | API | P0 | **FULL** |
| AC2 | Agent payload structure + X-Caller-Service header + timeout | `test_consortium_finder_agent_payload_has_required_fields`, `test_consortium_finder_x_caller_service_header_sent`, `test_consortium_finder_forwards_*` (2 tests) | API | P0 | **FULL** |
| AC3 | ConsortiumFinderResponse structure: partners, total_results, page, page_size | `TestAC3AC5ResponseParsing`: `test_partners_have_required_fields`, `test_total_results_and_page_size_match_partner_count` | API | P1 | **FULL** |
| AC4 | Single-page contract: page always 1, page_size == len(partners) | `test_total_results_and_page_size_match_partner_count` | API | P1 | **FULL** |
| AC5 | Partners sorted descending by collaboration_score | `test_partners_returned_in_descending_collaboration_score_order`, `test_capability_overlap_ranking_reflected_in_scores` | API | P1 | **FULL** |
| AC6 | Partial response graceful degradation: missing fields → null/[] not crash | `TestAC6GracefulDegradation` (4 tests): empty results, missing contact_info, missing past_projects, missing partners key | API | P2 | **FULL** |
| AC7 | Gateway timeout/5xx → 503 AGENT_UNAVAILABLE | `TestAC7AgentErrorHandling` (3 tests) | API | P0 | **FULL** |
| AC8 | Unauthenticated → 401; all roles permitted | `TestAC8Authorization` (2 tests) | API | P0 | **FULL** |
| AC9 | Stateless — no DB writes | Implicit via architecture (no AsyncSession dep) | API | — | **FULL** |

**Story Coverage:** 9/9 ACs → **FULL** | 26 tests total (P0: 9, P1: 7, P2: 10)

---

### S11.07 — Logframe Generator & Reporting Template Agent Integrations

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `POST /grants/logframe-generate` accepts project_narrative (required, min_length=1), optional target_programme; returns 200 | `TestAC1AC2HappyPath` (4 tests) + `TestAC1InputValidation` (2 tests) | API | P0 | **FULL** |
| AC2 | Agent payload: project_narrative, target_programme, company_id (str); X-Caller-Service; timeout 30s | `test_logframe_agent_payload_has_required_fields`, `test_logframe_x_caller_service_header_sent`, `test_logframe_null_target_programme_forwarded` | API | P0 | **FULL** |
| AC3 | LogframeResponse: all 4 fields (logical_framework, work_packages, gantt_data, deliverable_table) with sub-types | `TestAC3AC4ResponseParsing` (7 tests): all fields, each sub-type validated | API | P1 | **FULL** |
| AC4 | gantt_data: null when key absent; [] when present but empty (distinct!) | `test_gantt_data_absent_returns_null_not_error` (null) + `test_gantt_data_present_empty_list_not_null` (empty list distinct from null) | API | P1 | **FULL** |
| AC5 | Gateway timeout/5xx → 503 AGENT_UNAVAILABLE | `TestAC5AgentErrorHandling` (3 tests) | API | P0 | **FULL** |
| AC6 | Unauthenticated → 401; all roles allowed; stateless | `TestAC6Authorization` (1 test: unauthenticated → 401) | API | P0 | **FULL** |
| AC7 | `POST /reporting-template` accepts project_id (UUID, required); returns 200 or 404 | `TestAC7AC8HappyPath` (1 test: 200), `TestAC8NotFound` (1: 404), `TestInputValidation` (2: 422) | API | P0 | **FULL** |
| AC8 | Service loads project from DB with company RLS; builds agent payload; calls reporting-template-generator | `test_reporting_template_agent_payload_includes_project_data`, `test_reporting_template_x_caller_service_header_sent` | API | P1 | **FULL** |
| AC9 | ReportingTemplateResponse structure: project_id, milestones (list), sections (list) | `test_reporting_template_milestones_have_required_fields`, `test_reporting_template_sections_have_required_fields` | API | P1 | **FULL** |
| AC10 | Gateway timeout/5xx → 503 AGENT_UNAVAILABLE; 404 propagates unchanged | `TestAC10AgentErrorHandling` (3 tests): timeout, HTTP 500, export timeout | API | P0 | **FULL** |
| AC11 | `POST /reporting-template/export` returns DOCX; correct Content-Type + Content-Disposition | `TestAC11DOCX`: `test_export_returns_docx_content_type`, `test_export_has_content_disposition_attachment`, `test_export_body_is_non_empty`, `test_export_unknown_project_id_returns_404` | API | P1 | **FULL** |
| AC12 | DOCX includes heading, sections, milestones table, budget overview, consortium summary | `test_export_is_valid_word_document` (P2: python-docx parse + paragraphs > 0) | API | P2 | **PARTIAL** *(structure verified; content detail deferred to P3-006)* |

**Story Coverage:** 11/12 ACs FULL, 1/12 PARTIAL (AC12 structural only) | 35 tests total (P0: 14, P1: 16, P2: 5)

---

### S11.08 — Compliance Framework CRUD API (Admin)

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `POST /admin/compliance-frameworks` creates with name, description, country, regulation_type, rules JSONB | **No ATDD checklist** | — | P0/P1 | **NONE** |
| AC2 | `GET /admin/compliance-frameworks` lists with filters (country, regulation_type, is_active); paginated | **No ATDD checklist** | — | P1 | **NONE** |
| AC3 | `GET /admin/compliance-frameworks/:id` detail view | **No ATDD checklist** | — | P1 | **NONE** |
| AC4 | `PATCH /admin/compliance-frameworks/:id` updates any field | **No ATDD checklist** | — | P1 | **NONE** |
| AC5 | `DELETE /admin/compliance-frameworks/:id` soft-delete or hard-delete if unused | **No ATDD checklist** | — | P0/P1 | **NONE** |
| AC6 | Validate rules JSONB structure (rule_id, criterion, check_type, threshold, description) | **No ATDD checklist** | — | P2 | **NONE** |
| AC7 | Prevent deletion of frameworks currently assigned to active opportunities | **No ATDD checklist** | — | P0/P1 | **NONE** |
| AC8 | Enforce admin-only access via role-based middleware | **No ATDD checklist** | — | P0 | **NONE** |

**Story Coverage:** 0/8 ACs → **NONE** | 0 tests | ⚠️ ATDD checklist required

---

### S11.09 — Framework Assignment & Auto-Suggestion API (Admin)

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `POST /admin/opportunities/:id/compliance-frameworks` assigns one or more frameworks | **No ATDD checklist** | — | P1 | **NONE** |
| AC2 | `GET /admin/opportunities/:id/compliance-frameworks` lists assigned frameworks | **No ATDD checklist** | — | P1 | **NONE** |
| AC3 | `DELETE /admin/opportunities/:id/compliance-frameworks/:fid` removes assignment | **No ATDD checklist** | — | P1 | **NONE** |
| AC4 | Auto-suggestion flow: Framework Suggestion Agent invoked on opportunity ingest; suggestions stored with confidence scores | **No ATDD checklist** | — | P0/P1 | **NONE** |
| AC5 | `GET /admin/framework-suggestions` lists pending suggestions | **No ATDD checklist** | — | P0/P1 | **NONE** |
| AC6 | `PATCH /admin/framework-suggestions/:id` accept/reject; on accept → auto-assign atomically | **No ATDD checklist** | — | P1 | **NONE** |
| AC7 | Tests for assignment CRUD, suggestion generation, acceptance flow | **No ATDD checklist** | — | P1 | **NONE** |

**Story Coverage:** 0/7 ACs → **NONE** | 0 tests | ⚠️ ATDD checklist required

---

### S11.10 — Regulation Tracker Agent & Platform Settings API (Admin)

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Celery Beat task triggers Regulation Tracker Agent on configurable schedule; parses regulatory_changes | **No ATDD checklist** | — | P1 | **NONE** |
| AC2 | `GET /admin/regulatory-changes` lists with filters (status, severity, date range) | **No ATDD checklist** | — | P2 | **NONE** |
| AC3 | `PATCH /admin/regulatory-changes/:id` acknowledge/dismiss with optional notes | **No ATDD checklist** | — | P1 | **NONE** |
| AC4 | `GET /admin/platform-settings` admin only | **No ATDD checklist** | — | P2 | **NONE** |
| AC5 | `PATCH /admin/platform-settings/:key` manages configuration | **No ATDD checklist** | — | P2 | **NONE** |
| AC6 | Enforce admin-only access; tests for Celery task, CRUD, settings | **No ATDD checklist** | — | P1 | **NONE** |

**Story Coverage:** 0/6 ACs → **NONE** | 0 tests | ⚠️ ATDD checklist required

---

### S11.11 — EU Grant Tools Frontend — Eligibility & Budget Panels

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Grant Eligibility Panel: trigger button, loading spinner, results as programme list sorted by eligibility score | **No ATDD checklist** | — | P2 | **NONE** |
| AC2 | Budget Builder Panel: input form; results as editable table with cost categories, co-financing split; recalculated totals | **No ATDD checklist** | — | P2 | **NONE** |
| AC3 | Wire up React Query mutations; handle loading, error, empty states | **No ATDD checklist** | — | P2 | **NONE** |

**Story Coverage:** 0/3 ACs → **NONE** | 0 tests | ⚠️ ATDD checklist required (frontend/E2E)

---

### S11.12 — EU Grant Tools Frontend — Consortium Finder & Logframe Panels

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Consortium Finder Panel: tag input, multi-select countries, partner cards with collaboration score | **No ATDD checklist** | — | P2 | **NONE** |
| AC2 | Logframe Display Panel: logical framework table, work package cards, Gantt chart (Recharts), deliverable table | **No ATDD checklist** | — | P2 | **NONE** |
| AC3 | Reporting Template tab: project selector, generation, editable fields, Download DOCX button | **No ATDD checklist** | — | P2 | **NONE** |

**Story Coverage:** 0/3 ACs → **NONE** | 0 tests | ⚠️ ATDD checklist required (frontend/E2E)

---

### S11.13 — ESPD Profile Management & Auto-Fill Frontend

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | ESPD Profile List Page: table with profiles, create/edit/delete actions, empty state | **No ATDD checklist** | — | P2 | **NONE** |
| AC2 | ESPD Profile Editor: multi-step form mapped to ESPD Parts II-V with inline validation and help tooltips | **No ATDD checklist** | — | P2 | **NONE** |
| AC3 | ESPD Auto-Fill Page: side-by-side preview (original vs auto-filled), changed fields highlighted, accept/reject individual changes, XML/PDF download | **No ATDD checklist** | — | P2 | **NONE** |

**Story Coverage:** 0/3 ACs → **NONE** | 0 tests | ⚠️ ATDD checklist required (frontend/E2E)

---

### S11.14 — Compliance Admin — Framework Management Frontend

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Framework List Page: data table with name, country, regulation type, active status toggle, filters, pagination | **No ATDD checklist** | — | P2 | **NONE** |
| AC2 | Framework Editor Page: form with country dropdown, regulation type radio, dynamic rules editor (add/remove/reorder), preview panel | **No ATDD checklist** | — | P2 | **NONE** |

**Story Coverage:** 0/2 ACs → **NONE** | 0 tests | ⚠️ ATDD checklist required (frontend/E2E)

---

### S11.15 — Compliance Admin — Assignment, Suggestions & Regulation Tracker Frontend

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Framework Assignment Page: opportunity selector, assigned frameworks panel, add/remove assignments | **No ATDD checklist** | — | P2 | **NONE** |
| AC2 | Auto-Suggestion Queue: table of pending suggestions with confidence bars, accept/override/dismiss actions, batch processing | **No ATDD checklist** | — | P2 | **NONE** |
| AC3 | Regulation Tracker Dashboard: feed of detected changes, acknowledge/dismiss actions with notes, filter by status/severity/date | **No ATDD checklist** | — | P2 | **NONE** |

**Story Coverage:** 0/3 ACs → **NONE** | 0 tests | ⚠️ ATDD checklist required (frontend/E2E)

---

### S11.16 — E2E Integration Testing & Agent Error Handling Hardening

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | E2E Journey 1: company runs eligibility check → budget builder | **No ATDD checklist** | — | P3 | **NONE** |
| AC2 | E2E Journey 2: company creates ESPD profile → auto-fill → XML export | **No ATDD checklist** | — | P3 | **NONE** |
| AC3 | E2E Journey 3: admin creates framework → assigns → proposal validated (reuses E07 compliance checker) | **No ATDD checklist** | — | P3 | **NONE** |
| AC4 | E2E Journey 4: opportunity ingested → suggestion generated → admin accepts → framework assigned | **No ATDD checklist** | — | P3 | **NONE** |
| AC5 | E2E Journey 5: regulation tracker fires → admin views → acknowledges → reviews framework | **No ATDD checklist** | — | P3 | **NONE** |
| AC6 | Agent error hardening: standardised timeout (30s), retry logic (1 retry + exp backoff), graceful degradation, consistent error format across all E11 endpoints | **No ATDD checklist** *(partial coverage in S11.03–S11.07 for 6 agents)* | — | P0 | **PARTIAL** |

**Story Coverage:** 0/6 ACs FULL, 1 PARTIAL (AC6 via S11.03–S11.07) | 0 E2E tests | ⚠️ ATDD checklist required (fullstack/E2E)

---

## 4. Epic Test ID Coverage Matrix

### P0 — Critical (10 test IDs, 38 assertions required)

| Test ID | Requirement | Tests | TDD Phase | Coverage |
|---------|-------------|-------|-----------|----------|
| **E11-P0-001** | ESPD profile company RLS — Company A JWT cannot GET/PATCH/DELETE Company B profile (404, not 403) | S11.02: `test_company_a_cannot_get_company_b_profile_returns_404` (1), `test_company_a_cannot_patch_company_b_profile_returns_404` (1), `test_company_a_cannot_delete_company_b_profile_returns_404` (1); S11.03: `test_autofill_cross_company_returns_404` (1), `test_export_cross_company_returns_404` (1) | 🔴 RED | **FULL** (5 tests, all 3 RLS vectors) |
| **E11-P0-002** | ESPD Auto-Fill: timeout → structured 503 (not raw 500) | S11.03: `test_autofill_gateway_timeout_returns_503`, `test_autofill_gateway_500_returns_503_not_500`, `test_autofill_gateway_503_returns_503_with_standard_body` | 🔴 RED | **FULL** (3 tests: timeout + 500 + 503) |
| **E11-P0-003** | ESPD XML export validates against EU ESPD XSD (Parts II-V all present) | S11.03: `test_export_xml_is_well_formed`, `test_export_xml_root_has_correct_namespace`, `test_export_xml_contains_all_parts_present_in_espd_data` | 🔴 RED | **PARTIAL** *(structural + namespace checked; full lxml XSD conformance test scheduled weekly per test-design exit criteria)* |
| **E11-P0-004** | Grant Eligibility Agent: structured 503 on timeout; structured list returned on success | S11.04: `test_eligibility_check_returns_200_with_response_structure`, `test_gateway_timeout_returns_503_agent_unavailable` | 🔴 RED | **FULL** |
| **E11-P0-005** | Budget Builder: line items sum to total_budget; overhead correctly applied; co-financing split sums to total_requested_funding | S11.05: `test_valid_solo_budget_arithmetic_passes`, `test_inconsistent_line_item_total_returns_422`, `test_inconsistent_co_financing_sum_returns_422`, `test_overhead_inconsistency_returns_422` | 🔴 RED | **FULL** (4 tests: valid + 3 invalid) |
| **E11-P0-006** | Compliance Framework: non-admin JWT returns 403 on POST/GET/PATCH/DELETE /admin/compliance-frameworks | **No tests** (S11.08 has no ATDD checklist) | ⚫ NO TESTS | **NONE** — 🚨 BLOCKER |
| **E11-P0-007** | Framework Suggestion admin-only: company JWT returns 403 on GET/PATCH /admin/framework-suggestions | **No tests** (S11.09 has no ATDD checklist) | ⚫ NO TESTS | **NONE** — 🚨 BLOCKER |
| **E11-P0-008** | Agent error handling: all 8 E11 agent types return AGENT_UNAVAILABLE structure on agent 503 | S11.03 (ESPD Auto-Fill): 3 tests; S11.04 (Grant Eligibility): 3 tests; S11.05 (Budget Builder): 3 tests; S11.06 (Consortium Finder): 3 tests; S11.07 (Logframe + Reporting Template): 4+3 tests. **Missing:** Framework Suggestion (S11.09) + Regulation Tracker (S11.10) | 🔴 RED (6/8) | **PARTIAL** (6 of 8 agents covered) |
| **E11-P0-009** | AI Gateway mock: all 8 E11 agent types return deterministic fixture responses in CI | S11.03–S11.07: all 6 implemented agents have deterministic fixture responses. **Missing:** Framework Suggestion (S11.09) + Regulation Tracker (S11.10) | 🔴 RED (6/8) | **PARTIAL** (6 of 8 agents covered) |
| **E11-P0-010** | 30s timeout enforced on all E11 agent endpoints; request does not hang | S11.03: timeout test; S11.04: `test_gateway_timeout_returns_503_agent_unavailable`; S11.05: timeout test; S11.06: timeout test; S11.07: logframe + reporting template timeout tests. **Missing:** Framework Suggestion + Regulation Tracker | 🔴 RED (6/8) | **PARTIAL** (6 of 8 agents covered) |

**P0 Summary:** 4 FULL · 4 PARTIAL · 2 NONE (BLOCKERS) | P0 FULL%: **40%**

---

### P1 — High (18 test IDs, 42 assertions required)

| Test ID | Requirement | Tests | TDD Phase | Coverage |
|---------|-------------|-------|-----------|----------|
| **E11-P1-001** | Grant Eligibility: full-match, partial-match, no-match; filter params applied | S11.04: `TestAC3ResponseParsing` (4 tests), `test_eligibility_check_with_programme_type_filter`, `test_eligibility_check_with_funding_range_filter` | 🔴 RED | **FULL** |
| **E11-P1-002** | Budget Builder: per-partner breakdown present when consortium_size > 1; co_financing_split visualizable | S11.05: `TestAC5ConsortiumBudget` (4 tests) | 🔴 RED | **FULL** |
| **E11-P1-003** | Consortium Finder: paginated results; capability overlap ranking; single-country filter; max_results honoured | S11.06: `TestAC3AC5ResponseParsing` (6 tests): sorting, total_results, required fields, single-country, max_results, ranking | 🔴 RED | **FULL** |
| **E11-P1-004** | Logframe Generator: all 4 output fields present (logical_framework, work_packages, gantt_data, deliverable_table) | S11.07: `test_all_four_logframe_fields_present_in_response`, `test_logical_framework_rows_*`, `test_work_packages_*`, `test_gantt_tasks_*`, `test_deliverables_*` | 🔴 RED | **FULL** |
| **E11-P1-005** | Logframe: gantt_data absent → partial result with null gantt_data (no 500) | S11.07: `test_gantt_data_absent_returns_null_not_error` + `test_gantt_data_present_empty_list_not_null` | 🔴 RED | **FULL** |
| **E11-P1-006** | Reporting Template Generator: project data loaded from DB; agent called; pre-filled report returned as JSON | S11.07: `test_reporting_template_returns_200_with_json_structure`, `test_reporting_template_agent_payload_includes_project_data`, `test_reporting_template_x_caller_service_header_sent` | 🔴 RED | **FULL** |
| **E11-P1-007** | Reporting Template: DOCX export generated, correct Content-Type, download succeeds | S11.07: `test_export_returns_docx_content_type`, `test_export_has_content_disposition_attachment`, `test_export_body_is_non_empty` | 🔴 RED | **FULL** |
| **E11-P1-008** | ESPD CRUD: create, list, get, update, delete — all 5 endpoints functional | S11.02: `TestAC1`–`TestAC5` (smoke: each endpoint has a working test); S11.03: snapshot profile accessible via GET after auto-fill | 🔴 RED | **FULL** |
| **E11-P1-009** | ESPD espd_data validation: missing Part III → 422 with field detail | S11.02 notes: "Not in scope for S11.02; completeness enforced at export time in S11.03." S11.03: `TestAC10EmptyEspdValidation` covers empty espd_data → 422. S11.02: `test_post_invalid_part_type_returns_422` covers non-dict Part → 422. **Missing:** missing Part III specifically → 422 with field detail | 🔴 RED | **PARTIAL** *(empty + non-dict Part covered; specific missing Part III validation gap)* |
| **E11-P1-010** | Compliance Framework CRUD: create, list with filters, get, update, soft-delete (admin JWT) | **No tests** (S11.08 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P1-011** | Framework assignment: assign 1 framework to opportunity, list, remove | **No tests** (S11.09 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P1-012** | Hybrid assignment: assign 2 frameworks (national + EU) to same opportunity; list returns both | **No tests** (S11.09 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P1-013** | Framework auto-suggestion: Framework Suggestion Agent called on opportunity ingest; stored with confidence | **No tests** (S11.09 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P1-014** | Suggestion accept flow: PATCH → status=accepted + opportunity framework assignment created atomically | **No tests** (S11.09 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P1-015** | Suggestion reject flow: PATCH → status=rejected; no auto-assignment created | **No tests** (S11.09 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P1-016** | Regulation Tracker Celery task: fires with mocked agent; regulatory_changes records created | **No tests** (S11.10 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P1-017** | Regulation Tracker: acknowledge + dismiss flows; acknowledged change links to affected framework | **No tests** (S11.10 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P1-018** | Framework deletion guard: cannot DELETE framework assigned to active opportunity → 409 Conflict | **No tests** (S11.08 no ATDD) | ⚫ NO TESTS | **NONE** |

**P1 Summary:** 8 FULL · 1 PARTIAL · 9 NONE | P1 FULL%: **44.4%**

---

### P2 — Medium (20 test IDs, 37 assertions required)

| Test ID | Requirement | Tests | Phase | Coverage |
|---------|-------------|-------|-------|----------|
| **E11-P2-001** | Budget Builder: missing optional params → defaults applied or clear validation error | S11.05: `TestAC2OptionalParams` (6 tests) | 🔴 RED | **FULL** |
| **E11-P2-002** | Consortium Finder: empty results; contact_info absent → null, not crash | S11.06: `TestAC6GracefulDegradation` (4 tests) | 🔴 RED | **FULL** |
| **E11-P2-003** | Logframe: DOCX reporting export returns valid DOCX | S11.07: `test_export_is_valid_word_document` (python-docx parse) | 🔴 RED | **FULL** |
| **E11-P2-004** | ESPD CRUD cross-company: Company A cannot PATCH/DELETE Company B profile → 404 | S11.02: `test_company_a_cannot_patch_company_b_profile_returns_404`, `test_company_a_cannot_delete_company_b_profile_returns_404` | 🔴 RED | **FULL** |
| **E11-P2-005** | ESPD espd_data: all 4 Parts can be independently patched without overwriting other Parts | S11.02: `test_patch_espd_data_part_level_merge` (Part-level merge semantics) | 🔴 RED | **FULL** |
| **E11-P2-006** | Compliance Framework rules JSONB: invalid rule schema → 422 | **No tests** (S11.08 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-007** | Framework suggestion: override with alternative framework_id → override framework assigned | **No tests** (S11.09 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-008** | Regulation tracker: GET /admin/regulatory-changes filters (status, severity, date_range) functional | **No tests** (S11.10 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-009** | Platform settings: GET /admin/platform-settings (admin only); PATCH merges value | **No tests** (S11.10 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-010** | Platform settings: invalid key → 404; invalid JSON value → 422 | **No tests** (S11.10 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-011** | Grant Eligibility Panel E2E: loading spinner, error state on 503, empty state | **No tests** (S11.11 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-012** | Budget Builder Panel E2E: editable table cells recalculate totals on input | **No tests** (S11.11 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-013** | Consortium Finder Panel E2E: tag input + multi-select; partner card grid | **No tests** (S11.12 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-014** | Logframe Panel E2E: Gantt chart renders; deliverable table sortable | **No tests** (S11.12 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-015** | ESPD Profile List E2E: empty state; "Create New Profile" navigates | **No tests** (S11.13 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-016** | ESPD Profile Editor E2E: multi-step form + exclusion grounds checkboxes | **No tests** (S11.13 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-017** | ESPD Auto-Fill Preview E2E: side-by-side diff; download XML functional | **No tests** (S11.13 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-018** | Compliance Framework List E2E: filter + search + pagination | **No tests** (S11.14 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-019** | Framework Assignment Page E2E: add 2 frameworks; remove 1; list shows remaining | **No tests** (S11.15 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P2-020** | Auto-Suggestion Queue E2E: batch accept; confidence bar colour-coded; filter | **No tests** (S11.15 no ATDD) | ⚫ NO TESTS | **NONE** |

**P2 Summary:** 5 FULL · 0 PARTIAL · 15 NONE | P2 FULL%: **25%**

---

### P3 — Low (8 test IDs)

| Test ID | Requirement | Tests | Phase | Coverage |
|---------|-------------|-------|-------|----------|
| **E11-P3-001** | E2E Journey 1 (S11.16): eligibility check → budget builder | **No tests** (S11.16 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P3-002** | E2E Journey 2 (S11.16): ESPD profile → auto-fill → preview → XML export | **No tests** (S11.16 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P3-003** | E2E Journey 3 (S11.16): admin creates framework → assigns → proposal validated | **No tests** (S11.16 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P3-004** | E2E Journey 4 (S11.16): opportunity ingested → suggestion → admin accepts → auto-assign | **No tests** (S11.16 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P3-005** | E2E Journey 5 (S11.16): regulation tracker fires → admin acknowledges → reviews framework | **No tests** (S11.16 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P3-006** | Reporting Template DOCX: valid Word document with structured sections | S11.07: `test_export_is_valid_word_document` (P2/P3: python-docx parse, paragraphs > 0) | 🔴 RED | **PARTIAL** *(structural parse verified; full section/heading content deferred)* |
| **E11-P3-007** | Regulation Tracker Frontend E2E: feed renders; acknowledge/dismiss functional | **No tests** (S11.15/S11.16 no ATDD) | ⚫ NO TESTS | **NONE** |
| **E11-P3-008** | k6: 10 concurrent agent endpoint calls; p95 < 5s on mock; no 500s | **No tests** (S11.16 no ATDD) | ⚫ NO TESTS | **NONE** |

**P3 Summary:** 0 FULL · 1 PARTIAL · 7 NONE | P3 FULL%: **0%**

---

## 5. Coverage Heuristics Analysis

### Endpoint Coverage

| Category | Endpoints Exercised | Endpoints Without Tests |
|----------|--------------------|-----------------------|
| ESPD Profiles (S11.01–S11.03) | POST, GET(list), GET(detail), PATCH, DELETE `/espd-profiles/*`; POST `auto-fill`; POST `export` | None — all covered |
| Grant Tools (S11.04–S11.07) | POST `/grants/eligibility-check`; POST `/grants/budget-builder`; POST `/grants/consortium-finder`; POST `/grants/logframe-generate`; POST `/grants/reporting-template`; POST `/grants/reporting-template/export` | None — all covered |
| Compliance Admin (S11.08–S11.10) | — | POST/GET/PATCH/DELETE `/admin/compliance-frameworks/*`; POST/GET/DELETE `/admin/opportunities/:id/compliance-frameworks/*`; GET/PATCH `/admin/framework-suggestions/*`; GET/PATCH `/admin/regulatory-changes/*`; GET/PATCH `/admin/platform-settings/*` |
| Frontend Pages (S11.11–S11.15) | — | `/grants/tools`, `/espd`, `/admin/compliance`, `/admin/compliance/assign`, `/admin/compliance/suggestions`, `/admin/compliance/regulations` |

**Endpoints Without Tests:** 15+ admin endpoints (S11.08–S11.10) + all frontend pages (S11.11–S11.15)

### Auth / Authz Coverage

| Auth Scenario | Covered | Missing |
|---------------|---------|---------|
| ESPD RLS (company-scoped): 404 not 403 on cross-company | ✅ S11.02 + S11.03 (5 tests) | — |
| ESPD API auth: unauthenticated → 401 | ✅ S11.02 + S11.03 | — |
| ESPD API auth: low-privilege → 403 on write | ✅ S11.02 TestAC9 (13 parametrized cases) | — |
| Agent API auth: unauthenticated → 401 | ✅ S11.04, S11.05, S11.06, S11.07 | — |
| Agent API auth: all roles permitted (no role gate) | ✅ S11.04–S11.07: read_only role → 200 | — |
| Admin API auth: company JWT → 403 | ❌ **Missing** — S11.08 (framework CRUD), S11.09 (suggestions), S11.10 (tracker) | E11-P0-006, E11-P0-007 |
| Admin API auth: expired admin JWT → 401 | ❌ **Missing** — S11.08–S11.10 | E11-P0-006 (scope) |

### Error Path Coverage

| Error Scenario | Covered | Missing |
|----------------|---------|---------|
| AI Gateway timeout → 503 AGENT_UNAVAILABLE | ✅ S11.03–S11.07 (6 of 8 agents) | Framework Suggestion + Regulation Tracker |
| AI Gateway 500 → 503 (not raw 500) | ✅ S11.03–S11.07 | Framework Suggestion + Regulation Tracker |
| AI Gateway 503 → 503 with standard body | ✅ S11.03–S11.07 | Framework Suggestion + Regulation Tracker |
| Empty agent response graceful degradation | ✅ S11.04 (empty programmes list), S11.06 (missing keys) | — |
| Budget arithmetic inconsistency → 422 | ✅ S11.05 (3 inconsistency types) | — |
| Missing partner breakdown → 422 | ✅ S11.05 | — |
| Cross-company access → 404 (not 403) | ✅ S11.02 + S11.03 | — |
| Framework deletion guard → 409 | ❌ **Missing** — E11-P1-018 | S11.08 ATDD required |
| Suggestion accept atomicity failure | ❌ **Missing** — E11-R-007 | S11.09 ATDD required |
| Celery Beat task agent timeout | ❌ **Missing** — E11-R-006 | S11.10 ATDD required |

---

## 6. Gap Analysis

### Critical Gaps (P0 — Gate Blockers)

| Gap ID | Test ID | Story | Description | Risk |
|--------|---------|-------|-------------|------|
| **GAP-001** | E11-P0-006 | S11.08 | No tests for admin-only enforcement on compliance framework CRUD endpoints | E11-R-003 (SEC, score 6) |
| **GAP-002** | E11-P0-007 | S11.09 | No tests for admin-only enforcement on framework suggestion endpoints | E11-R-003 (SEC, score 6) |
| **GAP-003** | E11-P0-008 (partial) | S11.09, S11.10 | Framework Suggestion Agent + Regulation Tracker Agent error handling untested | E11-R-001 (TECH, score 6) |
| **GAP-004** | E11-P0-009 (partial) | S11.09, S11.10 | Framework Suggestion + Regulation Tracker not in CI smoke gate | E11-R-001 (TECH, score 6) |
| **GAP-005** | E11-P0-010 (partial) | S11.09, S11.10 | 30s timeout not verified for Framework Suggestion + Regulation Tracker | E11-R-001 (TECH, score 6) |
| **GAP-006** | E11-P0-003 (partial) | S11.03 | Full EU ESPD XSD conformance validation (lxml) not in per-PR suite | E11-R-002 (DATA, score 6) |

### High Gaps (P1 — Must Address Before Sprint Gate)

| Gap ID | Test IDs | Stories | Description |
|--------|----------|---------|-------------|
| **GAP-007** | E11-P1-010 | S11.08 | Zero coverage on Compliance Framework CRUD (admin): create, list, get, update, soft-delete |
| **GAP-008** | E11-P1-018 | S11.08 | Zero coverage on framework deletion guard (cannot delete if assigned to active opportunity) |
| **GAP-009** | E11-P1-011, E11-P1-012 | S11.09 | Zero coverage on framework assignment CRUD (single + hybrid national+EU) |
| **GAP-010** | E11-P1-013, E11-P1-014, E11-P1-015 | S11.09 | Zero coverage on auto-suggestion lifecycle (generate, accept, reject) |
| **GAP-011** | E11-P1-016, E11-P1-017 | S11.10 | Zero coverage on Regulation Tracker Celery task + acknowledge/dismiss flows |
| **GAP-012** | E11-P1-009 (partial) | S11.02/S11.03 | ESPD Part III missing specifically → 422 with field detail not fully tested |

### Medium Gaps (P2 — Frontend + Admin Edge Cases)

| Gap ID | Test IDs | Stories | Description |
|--------|----------|---------|-------------|
| **GAP-013** | E11-P2-006 | S11.08 | Rules JSONB validation (invalid schema → 422) |
| **GAP-014** | E11-P2-007 | S11.09 | Suggestion override with alternative framework_id |
| **GAP-015** | E11-P2-008, E11-P2-009, E11-P2-010 | S11.10 | Regulation tracker + platform settings edge cases |
| **GAP-016** | E11-P2-011, E11-P2-012 | S11.11 | Grant Eligibility + Budget Builder frontend panels (E2E) |
| **GAP-017** | E11-P2-013, E11-P2-014 | S11.12 | Consortium Finder + Logframe frontend panels (E2E) |
| **GAP-018** | E11-P2-015, E11-P2-016, E11-P2-017 | S11.13 | ESPD Profile management frontend (E2E) |
| **GAP-019** | E11-P2-018 | S11.14 | Compliance Framework List + Editor (E2E) |
| **GAP-020** | E11-P2-019, E11-P2-020 | S11.15 | Framework Assignment + Auto-Suggestion Queue (E2E) |

### Low Gaps (P3 — Deferred / Nightly)

| Gap ID | Test IDs | Stories | Description |
|--------|----------|---------|-------------|
| **GAP-021** | E11-P3-001 to E11-P3-005 | S11.16 | All 5 E2E integration journeys (eligibility→budget, ESPD, framework admin, suggestion, regulation tracker) |
| **GAP-022** | E11-P3-007 | S11.15/S11.16 | Regulation Tracker Frontend E2E |
| **GAP-023** | E11-P3-008 | S11.16 | k6 agent load test (10 concurrent, p95 < 5s) |

---

## 7. Phase 2 — Quality Gate Decision

### Gate Criteria Evaluation

| Criterion | Threshold | Actual | Status |
|-----------|-----------|--------|--------|
| P0 FULL coverage | **100%** | **40%** | ❌ NOT MET |
| P1 FULL coverage (PASS) | ≥ 90% | 44.4% | ❌ NOT MET |
| P1 FULL coverage (minimum) | ≥ 80% | 44.4% | ❌ NOT MET |
| Overall FULL coverage | ≥ 80% | 30.4% | ❌ NOT MET |

### Risk Context (from test-design-epic-11.md)

| Risk ID | Category | Score | Status | Mitigation Tests |
|---------|----------|-------|--------|-----------------|
| E11-R-001 | TECH — AI Gateway error handling (8 agents) | 6 | ⚠️ PARTIAL | 6/8 agents covered; Framework Suggestion + Regulation Tracker missing |
| E11-R-002 | DATA — ESPD XML schema non-conformance | 6 | ⚠️ PARTIAL | Structural tests written; full XSD weekly only |
| E11-R-003 | SEC — Admin endpoint authorization gaps | 6 | ❌ NONE | GAP-001, GAP-002 — zero coverage on admin auth |
| E11-R-004 | BUS — Budget arithmetic consistency | 6 | ✅ FULL | S11.05 covers all 3 inconsistency types |
| E11-R-005 | SEC — ESPD company-scoped RLS | 6 | ✅ FULL | S11.02 + S11.03: 5 RLS tests |
| E11-R-006 | OPS — Regulation Tracker Celery reliability | 4 | ❌ NONE | S11.10 ATDD required |
| E11-R-007 | DATA — Framework suggestion atomicity | 4 | ❌ NONE | S11.09 ATDD required |
| E11-R-008 | DATA — Logframe parser field completeness | 4 | ✅ FULL | S11.07 covers gantt_data null vs empty list |
| E11-R-009 | BUS — Framework deletion guard | 4 | ❌ NONE | S11.08 ATDD required |
| E11-R-010 | TECH — Hybrid framework assignment | 4 | ❌ NONE | S11.09 ATDD required |

### Blocking Issues

| Priority | Issue | Description | Story | Status |
|----------|-------|-------------|-------|--------|
| **P0** | GAP-001 — Admin auth zero coverage | No tests verify company JWT → 403 on `/admin/compliance-frameworks` | S11.08 | ⚫ OPEN |
| **P0** | GAP-002 — Suggestion auth zero coverage | No tests verify company JWT → 403 on `/admin/framework-suggestions` | S11.09 | ⚫ OPEN |
| **P0** | GAP-003/004/005 — 2 agent types not in CI gate | Framework Suggestion + Regulation Tracker have no mock responses, error handling, or timeout tests | S11.09, S11.10 | ⚫ OPEN |
| **P1** | GAP-007 to GAP-011 | Zero coverage on S11.08, S11.09, S11.10 admin backend stories (9 P1 test IDs) | S11.08–S11.10 | ⚫ OPEN |

### Recommendations

**[URGENT — Before Gate Can Move]**

1. **Run `bmad-testarch-atdd` for S11.08** — Compliance Framework CRUD API (Admin). Target: E11-P0-006, E11-P1-010, E11-P1-018, E11-P2-006 + admin auth tests for all framework endpoints.

2. **Run `bmad-testarch-atdd` for S11.09** — Framework Assignment & Auto-Suggestion. Target: E11-P0-007, E11-P1-011 through E11-P1-015, E11-P2-007 + admin auth tests for suggestion endpoints.

3. **Run `bmad-testarch-atdd` for S11.10** — Regulation Tracker & Platform Settings. Target: E11-P1-016, E11-P1-017, E11-P2-008 through E11-P2-010 + Celery Beat task tests.

**[HIGH — This Sprint]**

4. **Run `bmad-testarch-atdd` for S11.11–S11.15** — All 5 frontend stories (Playwright E2E). Target: E11-P2-011 through E11-P2-020.

5. **Run `bmad-testarch-atdd` for S11.16** — E2E Integration + Agent Error Hardening. Target: E11-P3-001 through E11-P3-008 + AC6 agent error hardening for Framework Suggestion + Regulation Tracker.

6. **Complete ESPD Part III validation gap** (GAP-012 / E11-P1-009) — Add specific test: `POST /espd-profiles` with espd_data missing `part_iii` → 422 with field-level error detail.

**[MEDIUM — Deferred / Nightly]**

7. **Add full EU ESPD XSD lxml validation** to per-PR suite (currently weekly-only per GAP-006). Risk E11-R-002 score 6 — consider promoting to per-PR for MVP milestone confidence.

8. **Run `bmad-testarch-atdd` for S11.16 k6 load test** — E11-P3-008: 10 concurrent agent calls, p95 < 5s on mock.

---

## 8. Sign-Off

**Phase 1 — Traceability Assessment:**

- Total Test IDs Tracked: 56 (P0: 10 · P1: 18 · P2: 20 · P3: 8)
- Total Tests Written: 210 (🔴 RED phase, S11.01–S11.07 only)
- Stories with Coverage: 7/16
- Stories Without Coverage: 9/16 (S11.08–S11.16)
- Overall FULL Coverage: 30.4% (17/56)
- Critical Gaps (P0): 6 (2 NONE + 4 PARTIAL)
- High Gaps (P1): 6 gap groups covering 9 test IDs

**Phase 2 — Gate Decision:**

- **Decision**: ❌ FAIL
- **P0 Evaluation**: ❌ ONE OR MORE FAILED (40% FULL, 2 blockers with zero coverage)
- **P1 Evaluation**: ❌ FAILED (44.4% FULL, 9 test IDs uncovered)
- **Overall Status**: ❌ FAIL

**Next Steps:**

- If FAIL ❌: Block deployment, run `bmad-testarch-atdd` for S11.08–S11.16, re-run this gate.
- Priority order: S11.08 → S11.09 → S11.10 → S11.11–S11.15 → S11.16

---

## TRACE_GATE: FAIL

**Generated:** 2026-04-11
**Workflow:** bmad-testarch-trace (Epic 11 — EU Grant Specialization & Compliance)

<!-- Powered by BMAD-CORE™ -->
