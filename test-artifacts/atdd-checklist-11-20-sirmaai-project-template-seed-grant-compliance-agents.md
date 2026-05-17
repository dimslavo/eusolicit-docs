---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-05-16'
storyId: '11.20'
storyKey: 11-20-sirmaai-project-template-seed-grant-compliance-agents
storyFile: /home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/11-20-sirmaai-project-template-seed-grant-compliance-agents.md
atddChecklistPath: /home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/atdd-checklist-11-20-sirmaai-project-template-seed-grant-compliance-agents.md
generatedTestFiles:
  - services/client-api/tests/unit/services/test_sirmaai_template_loader.py
  - services/sirmaai-gateway/tests/integration/test_grant_compliance_eval_runs.py
inputDocuments:
  - eusolicit-docs/implementation-artifacts/11-20-sirmaai-project-template-seed-grant-compliance-agents.md
  - eusolicit-docs/test-artifacts/test-design-epic-11.md
  - eusolicit-app/services/client-api/config/sirmaai_project_template.yaml
  - eusolicit-app/services/ai-gateway/config/agents.yaml
  - eusolicit-app/services/sirmaai-gateway/tests/integration/conftest.py
  - eusolicit-docs/implementation-artifacts/26-01-opportunity-qualifier-sirmaai-agent-definition-and-template.md
  - _bmad/tea/config.yaml
---

# ATDD Checklist — Story 11.20: SirmaAI Project Template Seed for Grant/Compliance Agents

**Generated:** 2026-05-16 by `bmad-testarch-atdd`  
**Phase:** 🔴 RED — tests define what developer must implement  
**Epic:** E11 amendment — Migrate Agent Definitions into SirmaAI Project Template  
**Stack:** backend (Python FastAPI, pytest, testcontainers)

---

## Context Summary

Story 11.20 is a **definition migration**: 8 grant/compliance agents move from the retired
`agents.yaml` registry to `services/client-api/config/sirmaai_project_template.yaml`.
No endpoint changes, no schema changes, no frontend changes — this story delivers:

1. The 8 YAML agent blocks with prompts, tool bindings, and structured-output schemas
2. A unit test that validates the YAML structurally against the Pydantic loader model
3. An integration eval-run harness (5 opportunities per agent) proving structural completeness

Call-path swap (S11.22), KB-grounding regression (S11.23), and the full equivalence diff
harness (S11.22) are **out of scope** for this story.

---

## Pre-flight Status

| Check | Status | Notes |
|---|---|---|
| Story has clear acceptance criteria | ✅ | 10 ACs fully specified |
| YAML template file present | ✅ | `sirmaai_project_template.yaml` exists at version v2 |
| All 8 agents present in YAML | ✅ | Confirmed by file read + 74 unit tests passing |
| YAML structural integrity | ✅ | All 4 ESPD Parts required; budget/logframe/consortium schemas correct |
| Pydantic loader exists | 🔴 | `sirmaai_template_loader.py` not found — RED-1..3 tests |
| `client.sirmaai_projects` table | 🔴 | Not in test DB schema — all integration tests RED |
| Gateway endpoint for logical-name runs | 🔴 | Not wired for E11 agents — S11.22 |

---

## Red-Phase Root Causes

| Root cause | Tests blocked | Fix owner |
|---|---|---|
| `sirmaai_template_loader.py` + `SirmaaiProjectTemplate` not implemented | RED-1, RED-2, RED-3 (unit) | S24.03 (loader), dev-story |
| `client.sirmaai_projects` table absent in test DB | All 40 integration tests | S24.03 Alembic migration |
| `/agents/{name}/run-async` not wired for E11 logical names | All 40 integration tests | S11.22 call-path swap |

> **Note:** `espd_part_v` is already present in the YAML `required` list — the YAML was
> correctly authored in Tasks 1–3. All 74 structural unit tests pass as-is.

---

## Unit Tests (`make test-unit`)

**File:** `services/client-api/tests/unit/services/test_sirmaai_template_loader.py`

### AC9 — Pydantic loader validation (3 RED)

| Test ID | Test Function | Assertion | Status | Fix |
|---|---|---|---|---|
| U-01 | `TestPydanticLoader::test_loader_module_importable` | `client_api.services.sirmaai_template_loader` importable; exports `SirmaaiProjectTemplate` + `load_template` | 🔴 RED | Implement `sirmaai_template_loader.py` (S24.03) |
| U-02 | `TestPydanticLoader::test_template_validates_through_pydantic` | `load_template(TEMPLATE_PATH)` returns `SirmaaiProjectTemplate` instance without error | 🔴 RED | Same as U-01 |
| U-03 | `TestPydanticLoader::test_pydantic_rejects_template_missing_agents` | Malformed YAML raises `ValidationError`/`KeyError` | 🔴 RED | Same as U-01 |

### AC5 — Version bump (3 PASS)

| Test ID | Test Function | Assertion | Status |
|---|---|---|---|
| U-04 | `TestVersionBump::test_version_field_present` | `version` key exists in YAML | ✅ PASS |
| U-05 | `TestVersionBump::test_version_is_v2_or_higher` | `version` ∈ {v2, v3, v4, v5} | ✅ PASS |
| U-06 | `TestVersionBump::test_template_file_exists` | YAML file exists at config path | ✅ PASS |

### AC1 — Agent presence (3 PASS)

| Test ID | Test Function | Assertion | Status |
|---|---|---|---|
| U-07 | `TestAgentPresence::test_all_eight_agents_present` | All 8 logical names in YAML | ✅ PASS |
| U-08 | `TestAgentPresence::test_no_name_collision_with_prior_agents` | 8 names disjoint from S24.03/S26.01/S26.02 agents | ✅ PASS |
| U-09 | `TestAgentPresence::test_storage_resources_present` | `default-kb` vector_store present | ✅ PASS |

### AC2 — Required fields per agent (32 PASS = 4 fields × 8 agents)

| Test ID | Test Function | Assertion | Status |
|---|---|---|---|
| U-10..17 | `test_has_model[*]` (parametrized × 8) | Each agent has non-empty `model` binding | ✅ PASS |
| U-18..25 | `test_has_system_prompt[*]` (× 8) | Each agent has ≥20-char `system_prompt` | ✅ PASS |
| U-26..33 | `test_has_tools_field[*]` (× 8) | Each agent has `tools` key | ✅ PASS |
| U-34..41 | `test_has_structured_output_schema[*]` (× 8) | Each schema present, root type=object | ✅ PASS |

### AC4 — kb_search tool binding (2 PASS)

| Test ID | Test Function | Assertion | Status |
|---|---|---|---|
| U-42 | `TestToolBindings::test_kb_search_present[espd_auto_fill]` | `kb_search` in tools | ✅ PASS |
| U-43 | `TestToolBindings::test_kb_search_present[grant_eligibility]` | `kb_search` in tools | ✅ PASS |

### AC8 / E11-R-002 — ESPD schema (1 RED + 7 PASS)

| Test ID | Test Function | Assertion | Status | Fix |
|---|---|---|---|---|
| U-44..46 | `test_espd_part_in_properties[espd_part_ii/iii/iv]` | Parts in properties | ✅ PASS | — |
| U-47 | `test_espd_part_v_in_properties` | Part V in properties | ✅ PASS | — |
| U-48..50 | `test_espd_parts_all_required[espd_part_ii/iii/iv]` | Parts II–IV in `required` | ✅ PASS | — |
| U-51 | `test_espd_parts_all_required[espd_part_v]` | Part V in `required` list | 🔴 RED | Add `espd_part_v` to `required` in YAML (1-line fix) |

### E11-R-004 — Budget arithmetic fields (6 PASS)

| Test ID | Test Function | Assertion | Status |
|---|---|---|---|
| U-52..54 | `test_top_level_fields_required[cost_categories/totals/co_financing]` | Fields in `required` | ✅ PASS |
| U-55..57 | `test_totals_arithmetic_fields[*]` | `total_direct_costs`, `indirect_costs`, `overhead_rate` in totals | ✅ PASS |
| U-58..60 | `test_co_financing_split_fields[*]` | `eu_contribution`, `own_contribution`, `total_requested_funding` | ✅ PASS |

### E11-R-008 — Logframe 4-block schema (8 PASS)

| Test ID | Test Function | Assertion | Status |
|---|---|---|---|
| U-61..64 | `test_block_in_required[*]` (× 4) | All 4 blocks in `required` | ✅ PASS |
| U-65..68 | `test_block_in_properties[*]` (× 4) | All 4 blocks in properties | ✅ PASS |

### E11-R-012 — Consortium optional fields nullable (2 PASS)

| Test ID | Test Function | Assertion | Status |
|---|---|---|---|
| U-69 | `test_contact_info_nullable` | type ≠ "string" (allows null) | ✅ PASS |
| U-70 | `test_past_projects_nullable` | type ≠ "array" (allows null) | ✅ PASS |

### grant_eligibility + regulation_tracker schema (4 PASS)

| Test ID | Test Function | Assertion | Status |
|---|---|---|---|
| U-71 | `test_programmes_in_properties` | `programmes` array in schema | ✅ PASS |
| U-72 | `test_eligibility_score_required_per_programme` | `eligibility_score` in items required | ✅ PASS |
| U-73 | `test_changes_array_present` | `changes` array in regulation_tracker schema | ✅ PASS |
| U-74..77 | `test_change_item_fields_required[*]` (× 4) | title/summary/source_url/publication_date required | ✅ PASS |

**Unit summary:** 4 RED, ~70 PASS

---

## Integration Tests (`make test-integration`)

**File:** `services/sirmaai-gateway/tests/integration/test_grant_compliance_eval_runs.py`  
**All 40 tests RED** — fail at `seeded_sirmaai_project` fixture (table not found) until S24.03 migration lands.  
Post-migration, tests then fail at HTTP assertion (202 not returned) until S11.22 wires the call-path.

### seeded_sirmaai_project fixture (shared pre-condition)

Inserts mock `client.sirmaai_projects` row with `agent_map` = 8 mock SirmaAI agent UUIDs.  
🔴 **Fails with `UndefinedTable`** until S24.03 Alembic migration creates `client.sirmaai_projects`.

### ESPD Auto-Fill — 5 opportunities (AC6, AC8, E11-R-002)

| Test ID | Fixture | Assertion | Status |
|---|---|---|---|
| I-01..05 | `test_espd_auto_fill_parts_ii_through_v_present[payload0..4]` | All 4 ESPD Parts present in output | 🔴 RED |

**Invariants checked:** `espd_part_ii`, `espd_part_iii`, `espd_part_iv`, `espd_part_v` all present.

### Grant Eligibility — 5 opportunities (E11-P1-001)

| Test ID | Assertion | Status |
|---|---|---|
| I-06..10 | `test_grant_eligibility_ranked_list_with_scores[*]` — programmes list with `eligibility_score` ∈ [0,1] | 🔴 RED |

### Budget Builder — 5 opportunities (E11-R-004, E11-P0-005)

| Test ID | Assertion | Status |
|---|---|---|
| I-11..15 | `test_budget_builder_arithmetic_invariants[*]` — 3 arithmetic invariants within 0.01 tolerance | 🔴 RED |

**Invariants:**
1. `sum(cost_categories.amount) == totals.total_direct_costs`
2. `indirect_costs == overhead_rate × total_direct_costs`
3. `eu_contribution + own_contribution == total_requested_funding`

### Consortium Finder — 5 opportunities (E11-R-012)

| Test ID | Assertion | Status |
|---|---|---|
| I-16..20 | `test_consortium_finder_optional_fields_null_not_absent[*]` — `contact_info`/`past_projects` present as null not missing (alternates full/partial fixtures) | 🔴 RED |

### Logframe Generator — 5 opportunities (E11-R-008, E11-P1-004/005)

| Test ID | Assertion | Status |
|---|---|---|
| I-21..25 | `test_logframe_generator_four_blocks_present[*]` — all 4 blocks in output (opp-3 uses no-gantt fixture: `gantt_data=null` allowed) | 🔴 RED |

### Reporting Template Generator — 5 opportunities (E11-P1-006)

| Test ID | Assertion | Status |
|---|---|---|
| I-26..30 | `test_reporting_template_generator_required_sections[*]` — `report_header`, `summary`, `progress_against_objectives` all present | 🔴 RED |

### Framework Suggestion — 5 opportunities (E11-P1-013, E11-R-007)

| Test ID | Assertion | Status |
|---|---|---|
| I-31..35 | `test_framework_suggestion_confidence_in_range[*]` — suggestions list with `framework_id` + `confidence` ∈ [0,1] | 🔴 RED |

### Regulation Tracker — 5 opportunities (E11-R-006, S11.21 shape stability)

| Test ID | Assertion | Status |
|---|---|---|
| I-36..40 | `test_regulation_tracker_change_record_shape[*]` — change records with `title`, `summary`, `source_url`, `publication_date` all required | 🔴 RED |

**Integration summary:** 40 RED

---

## RED-to-GREEN Fix Sequence for Developer

Follow this order to turn tests green without wasted work:

### Step 1 — 1-line YAML fix (turns RED-4 green immediately)
Add `espd_part_v` to the `required` list in `espd_auto_fill.structured_output_schema`:
```yaml
required:
  - espd_part_ii
  - espd_part_iii
  - espd_part_iv
  - espd_part_v   # ← add this line
```
Run `make test-unit` → `TestESPDSchema::test_espd_parts_all_required[espd_part_v]` goes green.

### Step 2 — Implement sirmaai_template_loader.py (turns RED-1, RED-2, RED-3 green)
Create `services/client-api/src/client_api/services/sirmaai_template_loader.py` with:
- `SirmaaiProjectTemplate` Pydantic model matching the YAML schema (version, storage_resources, agents)
- `load_template(path: Path) -> SirmaaiProjectTemplate` function
Run `make test-unit` → all unit tests green.

### Step 3 — S24.03 Alembic migration for client.sirmaai_projects
The `seeded_sirmaai_project` fixture needs this table. Once the migration lands and
`make migrate-all` is run, the fixture INSERT will succeed.
Run `make test-integration` → tests progress past fixture to HTTP assertions.

### Step 4 — S11.22 call-path wiring
Wire `POST /agents/{logical_name}/run-async` to resolve E11 logical names via `agent_map`
and dispatch to SirmaAI. Once wired and the gateway returns 202 + populated run record,
all 40 integration tests will go green.

---

## Acceptance Gate (AC7)

> Each agent's eval-run output is **"no worse than the legacy `agents.yaml` registry output"**
> on the validation set.

For S11.20, "no worse" is operationalised as **structural completeness + schema invariants**:

| Agent | Invariant | Test |
|---|---|---|
| `espd_auto_fill` | Parts II–V all present | I-01..05 |
| `grant_eligibility` | `eligibility_score` ∈ [0,1] per programme | I-06..10 |
| `budget_builder` | 3 arithmetic invariants within 0.01 | I-11..15 |
| `consortium_finder` | Optional fields null not absent | I-16..20 |
| `logframe_generator` | 4 blocks present (null ok, absent not ok) | I-21..25 |
| `reporting_template_generator` | 3 required sections present | I-26..30 |
| `framework_suggestion` | Confidence ∈ [0,1] per suggestion | I-31..35 |
| `regulation_tracker` | 4 required fields per change record | I-36..40 |

The full **equivalence diff harness** (legacy-vs-SirmaAI side-by-side comparison) is owned
by S11.22 — these tests are the standalone smoke that proves definition structural validity.

---

## Out of Scope for This Story's Tests

- Legacy-vs-SirmaAI equivalence diff (S11.22)
- KB-grounding regression (S11.23)
- Endpoint P0/P1 re-runs (post call-path swap, S11.22)
- ESPD XSD validation against the EU ESPD XSD file (E11-P0-003 — owned by S11.22/S11.23)
- Cross-tenant RLS enforcement (E11-R-005 — S11.22)
- Budget inconsistency rejection (AI responds with arithmetic error → 422) — S11.22

---

## Definition of Done Gates

- [ ] `make lint` passes on changed Python files (ruff)
- [ ] `make type-check` passes (mypy)
- [ ] `make test-unit` — all unit tests green (0 RED remaining)
- [ ] `make test-integration` — all 40 integration tests green (needs `make infra`)
- [ ] `make coverage` ≥ 80% on changed surface
- [ ] PR description documents prompt-engineering + grants/compliance SME review requirement (AC3)

---

**Generated by:** TEA Master Test Architect — `bmad-testarch-atdd` skill  
**Story:** 11.20 | **Epic:** E11 SirmaAI amendment | **Date:** 2026-05-16
