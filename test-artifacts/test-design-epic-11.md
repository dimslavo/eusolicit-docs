---
workflowStatus: 'completed'
totalSteps: 5
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
nextStep: ''
lastSaved: '2026-05-25'
inputDocuments:
  - /home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/epic-11-compliance-grants.md
  - /home/debian/Projects/eusolicit/eusolicit-docs/project-context.md
  - /home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/test-design-qa.md
  - /home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/test-design/eu-solicit-handoff.md
  - /home/debian/Projects/eusolicit/_bmad/bmm/config.yaml
  - .claude/skills/bmad-testarch-test-design/resources/knowledge/risk-governance.md
  - .claude/skills/bmad-testarch-test-design/resources/knowledge/probability-impact.md
  - .claude/skills/bmad-testarch-test-design/resources/knowledge/test-levels-framework.md
  - .claude/skills/bmad-testarch-test-design/resources/knowledge/test-priorities-matrix.md
---

# Test Design: Epic 11 - Compliance & Grants

**Date:** 2026-05-25
**Author:** Deb
**Status:** Draft
**Mode:** Epic-Level (Phase 4)
**Project:** EU Solicit

> **Related:** System-level plan (`test-design-qa.md`) and TEA→BMAD handoff (`test-design/eu-solicit-handoff.md`). This epic inherits system risks **R-006** (entity-level RBAC bypass), **R-018** (ESPD XML conformance), and **R-001** (AI Gateway resilience / mock mode). Epic-local risk IDs are prefixed `E11-R-`.

---

## Executive Summary

**Scope:** Epic-level test design for Epic 11 — three user-facing capabilities defined in the epic spec:

- **11.1 Run Compliance Check (ZOP):** validate a proposal against an assigned regulatory framework via the `compliance-checker` agent; return pass/fail/warning rules with severity; persist results; render interactive progress rings + accordion list + suggested remediations.
- **11.2 ESPD Generator:** auto-fill an ESPD from company profile + opportunity; produce schema-valid **XML and PDF**; drive a wizard stepper for fields requiring confirmation.
- **11.3 EU Grant Budget Calculator:** validate a budget against total caps, co-financing rules, and eligibility windows; perform float comparisons with `_ARITHMETIC_TOLERANCE = 0.01`.

These features are AI-gateway-backed (`sirmaai-gateway`/`ai-gateway`), company-scoped, and tier-gated (per the system plan). The dominant exposures are **regulatory/financial correctness** (a wrong pass/fail or budget conclusion has legal/funding consequences) and **cross-tenant isolation** (compliance results and ESPD profiles are sensitive company data).

**Grounding (verified in code):** `client-api/src/client_api/services/grants_service.py` (`_ARITHMETIC_TOLERANCE = 0.01`, `_validate_budget_arithmetic`, `422 BUDGET_ARITHMETIC_INCONSISTENT`), `espd_service.py` (`urn:X-eusolicit:espd:schema:v1`, `_sanitise_xml_tag`, XML/PDF/DOCX export), `api/v1/espd.py`. Existing tests: `test_budget_builder.py`, `test_espd_autofill_export.py`, `test_espd_profile.py`, `test_grant_eligibility.py`, `test_proposal_compliance_risk_scoring*.py`.

**Risk Summary:**

- Total risks identified: **13**
- High-priority risks (≥6): **7**
- Critical categories: **DATA** (arithmetic / XML / severity correctness), **SEC** (cross-tenant + tier), **BUS** (caps/windows), **TECH** (agent resilience, CPU-bound render)

**Coverage Summary:**

- P0 scenarios: **25** (~30–45 hours)
- P1 scenarios: **30** (~25–40 hours)
- P2/P3 scenarios: **31** (~15–30 hours)
- **Total effort**: ~**70–115 hours** (~2–3 weeks, 1 QA engineer)

---

## Not in Scope

| Item | Reasoning | Mitigation |
| --- | --- | --- |
| **Agent model quality** (relevance of `compliance-checker` / `budget-builder` outputs) | Third-party SirmaAI/KraftData model behaviour is outside application scope | Contract-mock the agent (respx); assert our **parsing, validation, persistence, and arithmetic guards**, not the model's judgement |
| **EU procurement portal acceptance of the ESPD XML** | Requires external portal sandbox access (out of CI) | Assert XML **well-formedness + namespace + mandatory Parts** against `urn:X-eusolicit:espd:schema:v1`; portal submission is a manual gate |
| **PDF/DOCX renderer internal fidelity** (pixel layout) | Library-owned rendering | Assert document **MIME, non-empty bytes, non-blocking generation**; visual fidelity is exploratory (P3) |
| **Stripe tier provisioning correctness** | Covered by Epic 8 billing tests | Reuse tier fixtures; this epic only asserts the **gate decision** (403 vs 200) on Epic 11 endpoints |
| **Stories 11.4–11.7** (eligibility, consortium finder, logframe, reporting template) | Beyond the 3-story epic spec; implementation has expanded into these | Tracked as a follow-up test-design increment; existing tests already cover much of that surface — `*trace` before adding new tests |

---

## Risk Assessment

### High-Priority Risks (Score ≥6) — MITIGATE before release

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **E11-R-09** | DATA | **Budget float arithmetic** — line-item sum, co-financing sum, overhead, and per-partner totals must reconcile within `_ARITHMETIC_TOLERANCE = 0.01`; a silent mismatch yields an ineligible budget presented as valid | 2 | 3 | **6** | Unit + API tests on `_validate_budget_arithmetic` across all four invariants; boundary cases at exactly ±0.01 | QA/Dev | Pre-merge |
| **E11-R-12** | DATA | **Inconsistent agent budget accepted** — agent returns a budget that fails arithmetic; service must `422 BUDGET_ARITHMETIC_INCONSISTENT`, never persist/return it | 2 | 3 | **6** | API test forcing each violation type → assert 422 + error code; assert no DB write (stateless) | QA | Pre-merge |
| **E11-R-10** | BUS | **Co-financing cap / eligibility window boundary** — at-cap, just-over-cap, and window edge (inclusive/exclusive) mis-validated → wrong eligibility | 2 | 3 | **6** | Parametrized boundary tests: total caps, EU co-financing rate ceiling, window start/end inclusivity | QA | Pre-merge |
| **E11-R-03** | DATA | **Compliance severity misclassification** — pass/fail/warning + severity from the agent mis-mapped or dropped → wrong legal conclusion shown to bid manager | 2 | 3 | **6** | API test asserting each severity round-trips into persisted result and response; malformed rule filtered, not silently passed | QA | Pre-merge |
| **E11-R-05** | DATA | **ESPD XML non-conformance** (system R-018) — generated XML not well-formed / missing namespace / missing mandatory Parts → rejected by EU portal | 2 | 3 | **6** | API+Unit: parse XML with `ElementTree`, assert root `<ESPDResponse xmlns=urn:...>`, mandatory Parts present, depth cap honoured | QA | Pre-merge |
| **E11-R-01** | SEC | **Cross-tenant compliance result access** (system R-006) — company B reads/triggers company A proposal compliance check | 2 | 3 | **6** | Negative test via `create_company_pair`: B → A `/{proposal_id}/compliance-check` GET+POST → 403/404; no ID leakage | QA | Pre-merge |
| **E11-R-06** | SEC | **Cross-tenant ESPD profile access** — company B generates/reads/exports company A ESPD profile | 2 | 3 | **6** | Negative test: B → A ESPD profile + XML/PDF export → 403/404; `check_entity_access()` honoured | QA | Pre-merge |

### Medium-Priority Risks (Score 3–5) — MONITOR

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **E11-R-02** | TECH | `compliance-checker` agent timeout/unavailable → must `503 AGENT_UNAVAILABLE`, no partial persist (system R-001) | 2 | 2 | 4 | respx-injected timeout/5xx → assert 503 + code; assert DB unchanged | QA |
| **E11-R-04** | BUS | Missing/stale `framework_id` → check runs with no criteria, silently "passes" | 2 | 2 | 4 | Test framework load path: criteria from `client.compliance_frameworks` reach agent payload; absent framework handled explicitly | QA |
| **E11-R-07** | TECH | PDF/XML generation is CPU-bound (python-docx/render) → blocks event loop or hangs without timeout | 2 | 2 | 4 | Assert generation runs via `run_in_executor`/thread; timeout path returns error, not hang | Dev |
| **E11-R-08** | SEC | **XML injection / XXE** — untrusted ESPD field content (`<`, `&`, entity refs) breaks XML or expands entities | 2 | 2 | 4 | Unit: field with markup/entity → escaped, `_sanitise_xml_tag` applied, no entity expansion on parse | QA |
| **E11-R-11** | DATA | Eligibility window timezone / inclusive-boundary handling | 2 | 2 | 4 | Unit boundary tests across UTC dates; assert window edge semantics documented and tested | QA |
| **E11-R-13** | SEC | Tier gating drift — compliance/ESPD/grants reachable below entitled tier | 2 | 2 | 4 | Parametrized tier-gate test (Free/Starter→403, Pro/Enterprise→200) reusing E12 `tier-gate` pattern | QA |

### Low-Priority Risks (Score 1–2) — DOCUMENT

| Risk ID | Category | Description | Probability | Impact | Score | Action |
| --- | --- | --- | --- | --- | --- | --- |
| **E11-R-14** | OPS | Compliance-check audit-log write failure must not 500 (non-blocking, fire-and-forget) | 1 | 2 | 2 | Monitor (covered by P2 audit assertion) |

### Risk Category Legend

- **TECH**: Technical/Architecture · **SEC**: Security · **PERF**: Performance · **DATA**: Data Integrity · **BUS**: Business Impact · **OPS**: Operations

---

## Entry Criteria

- [ ] AC for Stories 11.1–11.3 agreed by QA, Dev, PM
- [ ] `make infra` up (postgres + redis) and `make migrate-all` applied
- [ ] `compliance-checker`, `budget-builder`, ESPD agents mocked via **respx** (no live SirmaAI/KraftData credentials in CI) — system blocker TB-02 satisfied
- [ ] Factories ready: `CompanyFactory`, `UserFactory`, `ProposalFactory`, `OpportunityFactory`, ESPD-profile factory, `ComplianceFramework` seed helper
- [ ] `create_company_pair` + `register_and_verify_with_role` available for cross-tenant/tier tests
- [ ] Tier fixtures (Free/Starter/Professional/Enterprise) reusable from Epic 8/12

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95%; failures triaged with ticket)
- [ ] No open High (≥6) risk unmitigated — especially E11-R-09, E11-R-05, E11-R-01, E11-R-06
- [ ] Cross-tenant negative tests present for **every** company-scoped Epic 11 endpoint (compliance-check, ESPD profile/export, grant budget read paths)
- [ ] ESPD XML asserted well-formed + schema-shape valid; PDF/DOCX asserted valid MIME + non-blocking generation
- [ ] Budget arithmetic invariants (4 checks) covered with ±0.01 boundary cases
- [ ] `make lint`, `make type-check` clean; `make coverage` ≥ **80%** on changed surface
- [ ] Frontend: `pnpm lint && pnpm type-check`; `pnpm check:i18n` if new strings added

---

## Test Coverage Plan

> P0–P3 = **priority/risk level**, not execution timing. See Execution Order.

### P0 (Critical) — Run on every commit

**Criteria**: Blocks core journey + High risk (≥6) + No workaround + regulatory/financial correctness

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| --- | --- | --- | --- | --- | --- |
| 11.3 Budget arithmetic — all 4 invariants reconcile within ±0.01 (line-item, co-financing, overhead, per-partner) | Unit + API | E11-R-09 | 6 | QA/Dev | Parametrize each invariant; include exactly-at-tolerance boundary (`abs == 0.01`) |
| 11.3 Inconsistent agent budget → `422 BUDGET_ARITHMETIC_INCONSISTENT`, nothing persisted | API | E11-R-12 | 4 | QA | respx returns broken sums for each check; assert error code + stateless |
| 11.3 Total cap + co-financing + eligibility window boundary validation | API | E11-R-10 | 5 | QA | at-cap / over-cap / window-start / window-end / outside-window |
| 11.1 Compliance severity round-trip — pass/fail/warning + severity persisted and returned faithfully | API | E11-R-03 | 4 | QA | Assert each severity tier; malformed rule filtered, not counted as pass |
| 11.2 ESPD XML well-formed + schema-shape valid (root, namespace, mandatory Parts) | API + Unit | E11-R-05 | 4 | QA | `ElementTree.fromstring()` parses; assert `urn:X-eusolicit:espd:schema:v1`; mandatory Parts present |
| 11.1 Cross-tenant compliance access — company B → company A proposal check → 403/404 | API | E11-R-01 | 1 | QA | GET + POST; no proposal/company ID in error body |
| 11.2 Cross-tenant ESPD access — company B → company A ESPD profile + XML/PDF export → 403/404 | API | E11-R-06 | 1 | QA | `check_entity_access()`; cover read + both exports |

**Total P0**: **25** tests, **~30–45 hours**

### P1 (High) — Run on PR to main

**Criteria**: Important features + Medium risk (3–5) + common workflows

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| --- | --- | --- | --- | --- | --- |
| 11.1 Agent timeout/unavailable → `503 AGENT_UNAVAILABLE`, no partial persist | API | E11-R-02 | 3 | QA | respx timeout + 5xx; assert DB unchanged |
| 11.1 `framework_id` load — criteria from `client.compliance_frameworks` reach agent payload | API | E11-R-04 | 4 | QA | with framework / absent framework / framework of another company → 404 |
| 11.1 Persist + retrieve — `GET …/compliance-check` returns stored result; `{result: null}` when none | API | — | 3 | QA | Idempotent re-run overwrites prior result |
| 11.2 ESPD PDF/DOCX generation non-blocking (run_in_executor) + valid MIME + timeout path | API | E11-R-07 | 4 | QA/Dev | Assert thread-pool offload; `application/pdf`; bytes non-empty |
| 11.2 ESPD XML injection/XXE — markup + entity in field values escaped, no expansion | API + Unit | E11-R-08 | 3 | QA | `<`, `&`, `<!ENTITY>` → escaped; tag sanitised |
| 11.3 Consortium budget — `consortium_size>1` without per-partner → `422 MISSING_PARTNER_BREAKDOWN` | API | E11-R-12 | 2 | QA | Plus partner subtotal reconciliation |
| Tier gating on all Epic 11 endpoints (Free/Starter→403, Pro/Enterprise→200) | API | E11-R-13 | 4 | QA | Reuse `tier-gate-enforcement` pattern across compliance/espd/grants |
| 11.1 FE — progress rings + accordion render severity from `<QueryGuard>` state | Component | E11-R-03 | 4 | Dev | Loading/error/empty/populated; remediation text present |
| 11.2 FE — ESPD wizard stepper gates progression on fields requiring confirmation | Component | — | 3 | Dev | Cannot advance until confirmation; persists across reload (Zustand) |

**Total P1**: **30** tests, **~25–40 hours**

### P2 (Medium) — Run nightly

**Criteria**: Secondary flows + low/medium risk + edge cases + regression prevention

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| --- | --- | --- | --- | --- | --- |
| 11.1 Audit-log entry on compliance-check mutation; failure is non-blocking (no 500) | API | E11-R-14 | 3 | QA | Assert `action_type/entity_type/entity_id`; fire-and-forget |
| 11.1 Agent malformed response (non-dict rules / missing keys) filtered gracefully | Unit | E11-R-03 | 4 | QA | No `AttributeError`; counts exclude bad entries |
| 11.3 Float edge cases — rounding at 0.005, large magnitudes, negative clamping to 0 | Unit | E11-R-09 | 6 | Dev | `_parse_cost_category`/`_parse_partner_budget` clamps |
| 11.3 Eligibility window timezone / inclusive-boundary semantics | Unit | E11-R-11 | 4 | QA | UTC date edges |
| 11.2 ESPD XML tag sanitisation + depth cap | Unit | E11-R-05/08 | 4 | QA | Invalid key chars → safe tag; depth beyond cap truncated |
| 11.2 ESPD auto-fill field population from company profile + opportunity | API | — | 4 | QA | Mandatory fields populated; unknown fields flagged for confirmation |

**Total P2**: **25** tests, **~10–22 hours**

### P3 (Low) — Run on-demand / nightly E2E

**Criteria**: Nice-to-have, full-journey, i18n, visual

| Requirement | Test Level | Test Count | Owner | Notes |
| --- | --- | --- | --- | --- |
| 11.2 E2E — generate ESPD via wizard → download XML + PDF (happy path) | E2E (Playwright) | 2 | QA | Chromium gate |
| 11.1 E2E — run compliance check → rings + accordion + remediations visible | E2E (Playwright) | 2 | QA | Mocked agent |
| i18n — new compliance/grant/ESPD strings present in BG + EN | Component | 2 | Dev | `pnpm check:i18n` parity |

**Total P3**: **6** tests, **~5–8 hours**

> Execution timing (smoke → PR → nightly) is handled separately in **Execution Order** below; the P0–P3 labels above denote priority/risk only.

---

## Execution Order

### Smoke Tests (<5 min)
- [ ] Budget arithmetic happy path reconciles (API) — E11-R-09
- [ ] ESPD XML parses + has root namespace (Unit) — E11-R-05
- [ ] Compliance check returns persisted severity list (API) — E11-R-03

### P0 Tests (<10 min)
- [ ] Budget invariant boundary suite (Unit/API)
- [ ] Inconsistent-budget 422 suite (API)
- [ ] Total cap / co-financing / window boundary suite (API)
- [ ] Compliance severity round-trip (API)
- [ ] ESPD XML conformance (API/Unit)
- [ ] Cross-tenant compliance + ESPD negatives (API)

### P1 Tests (<30 min)
- [ ] Agent resilience (503) + framework load (API)
- [ ] PDF/DOCX non-blocking + XML injection (API/Unit)
- [ ] Tier gating matrix (API)
- [ ] FE progress rings + ESPD wizard (Component)

### P2/P3 Tests (nightly, <60 min)
- [ ] Audit, malformed-response, float-edge, window, tag-sanitisation (Unit/API)
- [ ] E2E ESPD + compliance journeys, i18n parity

**Execution model:** PR runs all functional tests (Unit/API/Component, target <15 min); E2E + full regression run nightly (Chromium gate).

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Total Hours (range) | Notes |
| --- | --- | --- | --- |
| P0 | 25 | ~30–45 | Arithmetic boundaries, XML parsing, cross-tenant fixtures |
| P1 | 30 | ~25–40 | respx resilience, tier matrix, component specs |
| P2 | 25 | ~10–22 | Unit edge cases, audit assertions |
| P3 | 6 | ~5–8 | Playwright E2E + i18n |
| **Total** | **86** | **~70–115** | **~2–3 weeks, 1 QA** |

### Prerequisites

**Test Data:** `CompanyFactory`, `UserFactory`, `ProposalFactory`, `OpportunityFactory`, ESPD-profile factory, `ComplianceFramework` seed helper, tier fixtures.

**Tooling:** pytest + pytest-asyncio, **respx** (mock `compliance-checker`/`budget-builder`/ESPD/`reporting-template-generator` agents), testcontainers (postgres/redis), `xml.etree.ElementTree` for XML assertions, Playwright (Chromium) for E2E, Vitest for component specs.

**Environment:** local `make infra`; CI with testcontainers; no live SirmaAI/KraftData credentials.

---

## Quality Gate Criteria

### Pass/Fail Thresholds
- **P0 pass rate**: 100% (no exceptions)
- **P1 pass rate**: ≥95% (waivers ticketed)
- **P2/P3 pass rate**: ≥90% (informational)
- **High-risk mitigations**: 100% complete (E11-R-01/03/05/06/09/10/12)

### Coverage Targets
- Critical paths (compliance check, ESPD gen, budget validation): ≥80%
- Security scenarios (cross-tenant, tier, XXE): **100%**
- Business/financial logic (arithmetic, caps, windows, severity): ≥80% (above default 70% — regulatory)
- Edge cases: ≥50%

### Non-Negotiable Requirements
- [ ] All P0 tests pass
- [ ] No high-risk (≥6) item unmitigated
- [ ] SEC tests (E11-R-01/06/08/13) pass 100%
- [ ] Every company-scoped endpoint has a cross-tenant negative test (project rule)
- [ ] HMAC/secret comparisons use `hmac.compare_digest` (N/A unless a new webhook surface is introduced)

---

## Mitigation Plans

### E11-R-09: Budget float arithmetic (Score 6)
**Strategy:** Drive `_validate_budget_arithmetic` directly (unit) and via the endpoint (API) for all four invariants; include cases where `abs(diff)` is just below/at/above `_ARITHMETIC_TOLERANCE = 0.01`. **Owner:** QA/Dev. **Timeline:** Pre-merge. **Status:** Planned. **Verification:** consistent budget → 200; any violation → 422 with `BUDGET_ARITHMETIC_INCONSISTENT`.

### E11-R-05: ESPD XML conformance (Score 6)
**Strategy:** Parse generated XML; assert root element, namespace URI, mandatory Parts, escaping, depth cap. **Owner:** QA. **Timeline:** Pre-merge. **Status:** Planned. **Verification:** `ElementTree.fromstring(xml)` never raises; required elements present.

### E11-R-01 / E11-R-06: Cross-tenant isolation (Score 6 each)
**Strategy:** `create_company_pair`; company B credentials against company A compliance result and ESPD profile/exports → 403/404, no ID leakage. **Owner:** QA. **Timeline:** Pre-merge. **Status:** Planned. **Verification:** non-200 on every cross path; system R-006 satisfied.

### E11-R-03: Compliance severity correctness (Score 6)
**Strategy:** Mock agent returning each severity; assert faithful persistence + response; malformed rules filtered. **Owner:** QA. **Timeline:** Pre-merge. **Status:** Planned. **Verification:** severity counts and remediation text round-trip.

### E11-R-10 / E11-R-12: Caps/windows + inconsistent budget rejection (Score 6 each)
**Strategy:** Parametrized boundary suite for caps, co-financing ceiling, and window inclusivity; respx-broken sums per invariant assert `422` and zero persistence. **Owner:** QA. **Timeline:** Pre-merge. **Status:** Planned. **Verification:** boundary edges resolve to documented eligibility; inconsistent budgets never returned to user.

---

## Assumptions and Dependencies

### Assumptions
1. Compliance/ESPD/grant endpoints are AI-gateway-backed and **mockable via respx** in CI (no live SirmaAI calls).
2. Compliance check persists per-proposal; ESPD/budget grant flows are largely **stateless** (matches `grants_service.py`; reporting-template reads project DB).
3. Tier gating applies to these endpoints (consistent with Epic 6/8/12); exact entitled tiers confirmed from `tier_access_policy`.

### Dependencies
1. `ComplianceFramework` seed helper + ESPD-profile factory — required before P0 compliance/ESPD work.
2. Tier fixtures from Epic 8/12 — required for E11-R-13.
3. respx agent contract fixtures for `compliance-checker`, `budget-builder`, ESPD, `reporting-template-generator` (system blocker TB-02).

### Risks to Plan
- **Risk:** Epic 11 implementation has expanded beyond the 3 epic-spec stories (`grants_service.py` covers 11.4–11.7: eligibility, consortium finder, logframe, reporting template). **Impact:** coverage scoped to the 3 spec stories may understate effort. **Contingency:** treat 11.4–11.7 as a follow-up test-design increment; existing `test_grant_eligibility.py`, `test_budget_builder.py`, `test_espd_autofill_export.py`, `test_espd_profile.py`, `test_proposal_compliance_risk_scoring*.py` already cover much of that surface and should be traced (`*trace`) before adding new tests.

---

## Follow-on Workflows (Manual)
- Run `*atdd` to generate failing P0 tests (arithmetic invariants, XML conformance, cross-tenant negatives).
- Run `*trace` to map every AC (11.1–11.3) to a test ID before dev queue (Epic 10 lesson: trace **before** implementation).
- Run `*automate` for broader coverage once specs land.

---

## Approval

**Test Design Approved By:**
- [ ] Product Manager: __ Date: __
- [ ] Tech Lead: __ Date: __
- [ ] QA Lead: __ Date: __

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
| --- | --- | --- |
| **client-api (8001)** | Hosts compliance-check, ESPD, grant budget endpoints + services | `make test-service SVC=client-api`; existing `test_espd_profile.py`, `test_espd_autofill_export.py`, `test_budget_builder.py`, `test_grant_eligibility.py`, `test_proposal_compliance_risk_scoring*.py` must stay green |
| **sirmaai-gateway / ai-gateway (8004)** | Routes `compliance-checker`, `budget-builder`, ESPD, reporting agents | Mocked via respx; assert exact `Authorization: Bearer` header on outbound calls (E04 lesson) |
| **shared.audit_log** | Compliance-check mutation writes audit entry | P2 audit assertion; non-blocking |
| **Next.js client (3000)** | Compliance rings/accordion, ESPD wizard | `pnpm lint && pnpm type-check`; `make test-e2e-chromium` for P3 journeys |

---

## Appendix

### Knowledge Base References
- `risk-governance.md` — risk classification + gate decisions
- `probability-impact.md` — P×I scoring (1–9), thresholds
- `test-levels-framework.md` — Unit/API/Component/E2E selection
- `test-priorities-matrix.md` — P0–P3 prioritization

### Related Documents
- Epic: `eusolicit-docs/planning-artifacts/epics/epic-11-compliance-grants.md`
- System test plan: `eusolicit-docs/test-artifacts/test-design-qa.md` (R-006, R-018)
- TEA→BMAD handoff: `eusolicit-docs/test-artifacts/test-design/eu-solicit-handoff.md`
- Project context: `eusolicit-docs/project-context.md`
- Implementation: `services/client-api/src/client_api/services/{grants_service,espd_service}.py`; routes `api/v1/{grants,espd,proposals}.py`

---

**Generated by**: BMad TEA Agent — Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 4.0 (BMad v6)
