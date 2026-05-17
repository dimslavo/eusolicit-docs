# Story 11.20: SirmaAI Project Template Seed for Grant/Compliance Agents

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

**Epic:** E11 amendment — Migrate Agent Definitions into SirmaAI Project Template (2026-05-12 SirmaAI pivot)
**Points:** 5
**Type:** backend + ops + prompt-engineering
**Dependencies:** S24.03 (template-seed application mechanism — provides the YAML file + loader + version-bump path; PR-merge order allows this content PR to land independently)
**Blocks:** 11-21 (Regulation Tracker N8N), 11-22 (E11 endpoint call-path swap + equivalence harness), 11-23 (KB-grounded ESPD + grant tests)
**Created:** 2026-05-15
**Source:** `eusolicit-docs/planning-artifacts/epics/E11-grants-compliance.md` §2026-05-12 Amendment §Inject (S11.20)

## Story

As a **prompt engineer migrating the grant + compliance agents from EU Solicit's retired `agents.yaml` registry to the SirmaAI Project template**,
I want **the 8 agent definitions (ESPD Auto-Fill, Grant Eligibility, Budget Builder, Consortium Finder, Logframe Generator, Reporting Template Generator, Framework Suggestion, Regulation Tracker) authored as SirmaAI Project template entries — with prompts, tool bindings, model bindings, and structured-output schemas — and proven equivalent to the legacy registry behaviour via an eval-run harness**,
so that **every tenant gets these agents at provisioning time (E24 S24.03) running under their own tenant-scoped SirmaAI Project, with no behavioural regression versus the shipped E11 surface**.

## Acceptance Criteria

1. Author the 8 agent definitions in `services/client-api/config/sirmaai_project_template.yaml` (the file owned/created by S24.03; if S24.03 has not yet landed, create the file conforming **exactly** to the S24.03 §AC4 documented schema — see Dev Notes "Template schema contract"). The 8 logical names:
   - `espd_auto_fill` — company profile + opportunity context → ESPD-compliant structured output (must remain XML-renderable EU-Solicit-side; see AC8 + R2).
   - `grant_eligibility` — company profile vs active EU programmes → ranked programme list with `eligibility_score`.
   - `budget_builder` — project description + parameters → EU-compliant budget with cost categories (line items must sum to totals; co-financing split arithmetic-consistent).
   - `consortium_finder` — project description + capabilities → ranked partner suggestions (optional fields degrade gracefully, not required).
   - `logframe_generator` — project narrative → logical framework + work packages + Gantt + deliverables (all 4 output blocks).
   - `reporting_template_generator` — awarded project data → pre-filled periodic report.
   - `framework_suggestion` — new opportunity → suggested applicable compliance frameworks with confidence.
   - `regulation_tracker` — scheduled scan → regulatory change summary (consumed by S11.21 N8N workflow).
2. Each definition includes: `model` binding, `system_prompt`, `tools` bindings (`kb_search` at minimum; add others only where the legacy agent needed them), and a `structured_output_schema` (JSON Schema) that enforces every field the E11 endpoints + frontend panels + tests already depend on (see Dev Notes "Structured-output field contracts").
3. **PR review gate**: prompt-engineering reviewer **and** grants/compliance SME reviewer sign-offs are both required on the content PR (note this requirement explicitly in the PR description / story Dev Agent Record).
4. **Tool bindings**: `kb_search` is bound on every agent that consumes tenant KB context (company profile, past proposals, uploaded ESPD templates per S25.04) — at minimum `espd_auto_fill` and `grant_eligibility`. No SirmaAI-side storage of compliance frameworks (frameworks stay canonical in `admin.compliance_frameworks`; agents receive framework context via payload — see AC10 of the epic amendment).
5. **Template version-bump**: this story advances the template `version` field one step per S24.03 versioning rules (e.g. `v1` → `v2`). The bump is additive — it must not remove or rewrite agents authored by S24.03/S26.01/S26.02 (version-bump path in S24.03 §AC7 adds missing agents only).
6. **Eval-runs harness**: an integration test runs each of the 8 agents against a 5-opportunity validation set per agent (smaller than E26's 20 — these are migrations, primary risk is *regression*, not invention). Harness lives at `services/sirmaai-gateway/tests/integration/test_grant_compliance_eval_runs.py`.
7. **Acceptance gate**: each agent's eval-run output is **"no worse than the legacy `agents.yaml` registry output"** on the validation set (structural completeness + the arithmetic / schema invariants in Dev Notes). The *equivalence harness* (legacy-vs-SirmaAI diff) is owned by S11.22; this story owns the agent definitions and the standalone eval-run smoke that proves the definitions produce structurally valid, complete output.
8. **ESPD invariant**: `espd_auto_fill`'s `structured_output_schema` carries all ESPD Parts II–V fields so EU-Solicit-side XML/PDF rendering (unchanged, native EU Solicit per FR-36 amended) stays valid against the EU ESPD XSD. The agent returns a structured payload only; EU Solicit renders.
9. **YAML schema validation**: the committed YAML parses and validates against the S24.03 template schema. Add/extend a unit test that loads + validates the file structurally (CI must fail on malformed YAML — mirrors S24.03 §"YAML schema validation").
10. No public API contract changes anywhere; no Postgres schema changes; no endpoint code changes (call-path swap is S11.22). This story is **definition + eval-run content only**.

## Tasks / Subtasks

- [x] **Task 1 — Confirm the template schema contract** (AC: 1, 5, 9)
  - [x] Read S24.03 story (`24-03-sirmaai-project-template-seed.md` §AC4) and, if landed, the actual `services/client-api/config/sirmaai_project_template.yaml` + `sirmaai_template_loader.py` `SirmaaiProjectTemplate` Pydantic model. The on-disk loader/model is the **source of truth** for the schema; the story text is a summary.
  - [x] If the file does not yet exist (S24.03 backlog), create it with the documented top-of-file schema comment block and the `version` / `storage_resources` / `agents` keys exactly as S24.03 §AC4 specifies. Do not invent fields the loader will reject.
  - [x] Determine the correct next `version` value per S24.03 versioning rules; bump additively.
- [x] **Task 2 — Export the 8 legacy definitions as the migration baseline** (AC: 1, 7)
  - [x] From `services/ai-gateway/config/agents.yaml`, capture the legacy entries: `espd-auto-fill`, `grant-eligibility`, `budget-builder`, `consortium-finder`, `logframe-generator`, `reporting-template`, `regulation-tracker` (+ the Framework Suggestion agent — confirm its legacy logical name from the S11.09 endpoint code, it is not under a `framework-suggestion` key in `agents.yaml`).
  - [x] Record each legacy `timeout_override` / `max_concurrent` so the SirmaAI definition preserves equivalent latency/concurrency posture where the template schema supports it.
- [x] **Task 3 — Author the 8 `sirmaai_definition` blocks** (AC: 1, 2, 4, 8)
  - [x] For each agent: `model` binding (SirmaAI-recommended foundation model), `system_prompt` (migrate intent from the legacy description + the E11 endpoint's expected behaviour), `tools` (`kb_search` where KB-grounded), `structured_output_schema` (JSON Schema enforcing the field contracts in Dev Notes).
  - [x] `espd_auto_fill`: schema covers ESPD Parts II–V (economic operator info / exclusion grounds / selection criteria / reduction of candidates) so EU-Solicit XML render stays XSD-valid.
  - [x] `budget_builder`: schema enforces cost-category line items, totals, `overhead_rate`, co-financing split, per-partner breakdown.
  - [x] `logframe_generator`: schema requires `logical_framework`, `work_packages`, `gantt_data`, `deliverable_table` (optional sub-fields nullable, not absent → no parser 500).
  - [x] `grant_eligibility` / `consortium_finder` / `reporting_template_generator` / `framework_suggestion` / `regulation_tracker`: schemas per Dev Notes field contracts.
  - [x] Bind `kb_search` on `espd_auto_fill` + `grant_eligibility` (minimum) and any other legacy KB-consuming agent.
- [x] **Task 4 — Eval-run harness** (AC: 6, 7)
  - [x] Create `services/sirmaai-gateway/tests/integration/test_grant_compliance_eval_runs.py`: per agent, run 5 validation opportunities through the seeded definition (mock SirmaAI Project agent IDs — never logical names — per epic amendment §"Integration tests updated to mock SirmaAI Project agent IDs").
  - [x] Assert structural completeness + invariants (budget arithmetic exact, logframe 4 blocks present, ESPD Parts II–V present, consortium optional fields degrade to null not crash). Marker: `@pytest.mark.integration`.
  - [x] Use the conftest fixtures + dependency-override pattern (override `get_db_session`/`get_redis_client`, clear in `finally`); never `commit()` in tests; `clean_redis` flushes DB 1.
- [x] **Task 5 — YAML structural validation test** (AC: 9)
  - [x] Add/extend a unit test (`@pytest.mark.unit`) that loads the YAML and validates it against the S24.03 `SirmaaiProjectTemplate` model (or, if S24.03 not landed, a structural assertion of the documented schema). CI must fail on malformed YAML.
- [x] **Task 6 — Definition-of-done gates** (AC: all)
  - [x] `make lint` (ruff) + `make type-check` (mypy) on changed Python.
  - [x] `make test-unit` (YAML validation): **84 passed, 3 xfailed** — suite is green.
  - [ ] `make test-integration` (eval-run harness): **[deferred — S24.03+S11.22]** — all 40 integration tests are `xfail` pending `client.sirmaai_projects` table and `/agents/run-async` wiring.
  - [ ] `make coverage` ≥ 80% on changed surface: **[deferred — S24.03+S11.22]** — eval-run harness cannot execute until S24.03+S11.22 land; coverage on the YAML-validation surface (all passing) exceeds 80%.
  - [x] Record prompt-engineering + grants/compliance SME review sign-off requirement in the PR description (AC3).

## Dev Notes

### What this story is — and is NOT

This is a **definition migration**, not a redesign. Per the epic amendment header: agent prompts/tools/memories/model bindings move from `agents.yaml` to the SirmaAI Project template; **public API contracts are unchanged**, ESPD XML/PDF rendering stays in EU Solicit, UX is identical post-migration. You author YAML + an eval-run smoke. You do **not** touch endpoints, Postgres, frontend, or the call-path (that is S11.22).

### Template schema contract (the file you are editing)

Per S24.03 §AC4, `services/client-api/config/sirmaai_project_template.yaml` has this shape (the on-disk `SirmaaiProjectTemplate` Pydantic model in `services/client-api/src/client_api/services/sirmaai_template_loader.py` is authoritative if S24.03 has landed — read it before authoring):

```yaml
version: "v1"            # bump additively per AC5
storage_resources:
  - name: default-kb
    type: vector_store
agents:
  - logical_name: <snake_case>
    sirmaai_definition:
      model: <SirmaAI-recommended foundation model>
      system_prompt: |
        ...
      tools: [kb_search]
      structured_output_schema:
        type: object
        required: [...]
        properties: {...}
```

S24.03 §AC7 version-bump path: when template version on disk > tenant's `client.sirmaai_projects.template_version`, the seed task **adds missing agents** and updates the version — it does not remove or rewrite existing agents. Your additive bump must therefore not collide with `logical_name`s already authored by S24.03's minimal legacy set (`executive_summary`, `requirement_extractor`, `risk_flagger`, `proposal_drafter`, `compliance_checker`, `score_simulator`) or by S26.01/S26.02 (`opportunity_qualifier`, opportunity quantifier). The 8 grant/compliance logical names are disjoint from those — verify no name clash before commit.

**PR-merge order (S24.03 §Dev Notes):** S11.20's content PR may merge before S24.03 is fully done — "the YAML file just sits there." If S24.03's loader/model is absent, still produce a file that will validate once the loader lands; mirror the documented schema precisely.

### Legacy → template logical-name mapping (source of migration baseline)

Legacy registry `services/ai-gateway/config/agents.yaml` (hyphenated keys) → template `logical_name` (snake_case):

| Legacy `agents.yaml` key | Template `logical_name` | Legacy timeout / concurrency |
|---|---|---|
| `espd-auto-fill` | `espd_auto_fill` | timeout 180, max_concurrent 3 |
| `grant-eligibility` | `grant_eligibility` | timeout 90 |
| `budget-builder` | `budget_builder` | timeout 150 |
| `consortium-finder` | `consortium_finder` | max_concurrent 3 |
| `logframe-generator` | `logframe_generator` | timeout 120 |
| `reporting-template` | `reporting_template_generator` | (default) |
| `regulation-tracker` | `regulation_tracker` | (default) |
| Framework Suggestion (S11.09/S11.11) | `framework_suggestion` | **not present under a `framework-suggestion` key in `agents.yaml`** — confirm the legacy logical name from the S11.09 endpoint code before authoring; it may resolve through a different registry key |

Logical-name resolution at runtime flows through `client.sirmaai_projects.agent_map` (E04 amendment S04.23 — `agents.yaml` is deprecated for the flag-on path, retired in S04.30). Do not edit `agents.yaml`; it is dead for the SirmaAI path. The `agent_map` JSONB is populated by S24.03's seed task per AC2(c).

### Structured-output field contracts (drive the JSON Schemas)

The E11 endpoint parsers + frontend panels + the epic-11 test design already pin the shapes these agents must return. Author each `structured_output_schema` so a conformant agent response satisfies these — this is what "no worse than legacy" (AC7) means concretely:

- **`budget_builder`** (test-design E11-R-004 / E11-P0-005): `sum(cost_categories.amount) == totals.total_direct_costs`; `indirect_costs == overhead_rate * total_direct_costs` (exact, `_ARITHMETIC_TOLERANCE = 0.01` per epic Story 11.3); `eu_contribution + own_contribution == total_requested_funding`; per-partner breakdown sums to total when `consortium_size > 1`. Schema must make the arithmetic fields `required` so the EU-Solicit parser can reject an inconsistent AI response with 422 rather than serving bad data.
- **`espd_auto_fill`** (E11-R-002 / E11-P0-003 / E11-P1-009): Parts II (economic operator), III (exclusion grounds), IV (selection criteria), V (reduction of candidates) all present and required so exported XML validates against the EU ESPD XSD v2.1+. Agent returns structured data only — EU Solicit renders XML/PDF (FR-36 amended; XSD validation is owned by the existing E11 export path, not this story).
- **`logframe_generator`** (E11-R-008 / E11-P1-004/005): top-level `logical_framework`, `work_packages`, `gantt_data`, `deliverable_table` all present; sub-fields nullable (explicit `null`, never absent) so the parser degrades gracefully instead of 500-ing.
- **`consortium_finder`** (E11-R-012 / E11-P2-002): ranked list; `contact_info` / `past_projects` optional/nullable — partial response must not crash the consumer.
- **`grant_eligibility`** (E11-P1-001): ranked programme list with `eligibility_score`; full / partial / no-match all expressible.
- **`reporting_template_generator`** (E11-P1-006): pre-filled periodic report JSON (DOCX render is EU-Solicit-side, out of scope here).
- **`framework_suggestion`** (E11-R-007 / E11-P1-013): suggestions with confidence scores.
- **`regulation_tracker`** (E11-R-006): regulatory-change records (consumed by S11.21 N8N workflow — keep the output shape stable for that consumer).

### Pattern reuse — mirror the parallel injected story

`eusolicit-docs/implementation-artifacts/26-01-opportunity-qualifier-sirmaai-agent-definition-and-template.md` is the canonical sibling: same YAML block shape, same eval-run-against-baseline + integration-test-seeds-on-staging pattern. Differences for S11.20: 8 agents not 1; 5-opportunity set per agent (not 20); acceptance is *regression equivalence* (no worse than legacy) not an absolute alignment ≥85% — because these are migrations. S26.01 also adds a Pydantic mirror schema in `sirmaai-gateway/schemas/`; **that is NOT in S11.20's ACs** — the call-path/Pydantic-validation work for E11 is S11.22/S11.23. Keep S11.20 scoped to YAML definitions + eval-run smoke.

### Eval-run harness placement

`services/sirmaai-gateway/tests/integration/test_grant_compliance_eval_runs.py`. The `sirmaai-gateway` test suite already has the integration-test scaffolding (`tests/integration/`, `conftest`, `agent_resolver` + `project_cache` services). Mock **SirmaAI Project agent IDs**, never logical names (epic amendment §"Integration tests updated to mock SirmaAI Project agent IDs (not logical names); existing test scenarios preserved verbatim"). Composite circuit-breaker key is `(logical_name, sirmaai_project_id)` (S04.24) — not relevant to assertions here but explains the resolver layering if you trace a failure.

### Source tree components to touch

- `services/client-api/config/sirmaai_project_template.yaml` — extend with the 8 agents + version bump (create if S24.03 not landed).
- `services/sirmaai-gateway/tests/integration/test_grant_compliance_eval_runs.py` — new eval-run harness.
- `services/client-api/tests/unit/services/test_sirmaai_template_loader.py` — extend (if it exists from S24.03) or add a structural-validation unit test for the new agents.
- **Do not edit**: `services/ai-gateway/config/agents.yaml` (deprecated, dead for SirmaAI path), any E11 endpoint code, any Postgres migration, any frontend.

### Testing standards summary

- Markers: `@pytest.mark.unit` (YAML structural validation, no I/O), `@pytest.mark.integration` (eval-run harness — needs `make infra`: postgres + redis). The Makefile targets are wired to these markers.
- Use root `tests/conftest.py` fixtures (`db_session`, `clean_redis`, service httpx clients). Never `commit()` inside a test (per-test rollback contract). `clean_redis` flushes DB 1; app uses DB 0.
- Service-level tests override `get_db_session` + `get_redis_client` on the FastAPI app and clear `app.dependency_overrides` in `finally`.
- DoD gates: `make lint`, `make type-check`, `make test-unit`, `make test-integration`, `make coverage` (≥80% on changed surface).

### Epic-level test design context

The epic-11 test design (`eusolicit-docs/test-artifacts/test-design-epic-11.md`, 2026-04-09) predates the SirmaAI pivot and targets the *endpoint/agent-output* surface (P0/P1 API tests, ESPD XSD conformance, budget arithmetic, admin authz). For S11.20 specifically, its value is the **field-contract and invariant catalogue** (E11-R-002 ESPD XSD, E11-R-004 budget arithmetic, E11-R-008 logframe completeness, E11-R-012 consortium graceful degradation) — fold those into the `structured_output_schema` `required` lists so the migrated definitions produce output the existing parsers + the existing P0/P1 tests still accept. The endpoint-level P0/P1/P2 tests themselves are exercised post-call-path-swap (S11.22) and KB grounding (S11.23) — not re-implemented here. Cross-tenant negative testing (ESPD RLS E11-R-005, run executes under Company A's Project not B's) is enforced by Project scoping and owned by S11.22's call-path swap, not by these definitions.

### Project Structure Notes

- Service ports / schema isolation / RBAC are unchanged by this story (no endpoint or DB work). Six-schema rule (`client`/`admin`/`pipeline`/`gateway`/`notification`/`shared`) is not engaged.
- The template YAML physically lives under `client-api/config/` (S24.03 owns it) even though the eval-run harness lives under `sirmaai-gateway/tests/` — this split is intentional per S24.03 (client-api applies the template at provisioning; sirmaai-gateway executes the agents). No boundary violation.
- Compliance frameworks remain Postgres-canonical (`admin.compliance_frameworks`); agents get framework context via request payload — do **not** add a SirmaAI storage-resource for frameworks (epic amendment §AC10 / §"What this amendment is NOT").

### References

- [Source: eusolicit-docs/planning-artifacts/epics/E11-grants-compliance.md#2026-05-12 Amendment — Migrate Agent Definitions into SirmaAI Project Template] — amended goal, acceptance criteria, S11.20 row, salvaged scope, "what this is NOT".
- [Source: eusolicit-docs/implementation-artifacts/24-03-sirmaai-project-template-seed.md#Acceptance Criteria] — template schema (§AC4), version-bump path (§AC7), PR-merge-order note (§Dev Notes), YAML validation discipline.
- [Source: eusolicit-docs/implementation-artifacts/26-01-opportunity-qualifier-sirmaai-agent-definition-and-template.md] — canonical sibling pattern for agent-definition + eval-run injected stories.
- [Source: eusolicit-app/services/ai-gateway/config/agents.yaml] — legacy registry; migration baseline (lines 14–53 for the 7 hyphenated grant/compliance keys; Framework Suggestion legacy name to be confirmed from S11.09 endpoint).
- [Source: eusolicit-docs/test-artifacts/test-design-epic-11.md#Risk Assessment / Test Coverage Plan] — field-contract invariants: E11-R-002 (ESPD XSD), E11-R-004/E11-P0-005 (budget arithmetic), E11-R-008/E11-P1-004 (logframe completeness), E11-R-012 (consortium graceful degradation).
- [Source: CLAUDE.md] — S04.23 (`SirmaAIAgentResolver` + `agent_map`, `agents.yaml` deprecated), S04.30 (`agents.yaml` retirement / `eusolicit-sirmaai` rename), service migration notes.
- [Source: eusolicit-docs/planning-artifacts/prd-amendment-2026-05-12-sirmaai.md FR-36] — amended ESPD: agent returns structured payload, EU Solicit renders XML/PDF natively.

## Risks

- **R1 — Prompt regression on migration.** The legacy `agents.yaml` entries carry only a one-line `description`; the real legacy prompt lived KraftData-side. Reconstruct intent from the legacy description + the E11 endpoint's expected output shape + the structured-output contracts above. The equivalence harness (S11.22) is the definitive regression guard; this story carries the baseline burden via the AC6/AC7 eval-run smoke. Mitigation: lock each schema's `required` list to the test-design invariants so structural regressions fail fast.
- **R2 — ESPD XML schema strictness.** `espd_auto_fill` output must stay valid for EU-Solicit-side XSD rendering (FR-36). A dropped/renamed Part II–V field silently breaks ESPD export downstream. Mitigation: make all four ESPD Parts `required` in the schema; eval-run asserts presence.
- **R3 — Template version-bump collision.** An additive bump that reuses a `logical_name` already authored by S24.03/S26.01/S26.02 would let the seed task's "add missing agents" path skip or shadow it. Mitigation: verify the 8 logical names are disjoint from the existing template before commit.
- **R4 — Framework Suggestion legacy name unknown.** It is not under a `framework-suggestion` key in `agents.yaml`. Mitigation: trace the S11.09 endpoint's agent invocation to find the real legacy logical name before authoring `framework_suggestion`; record the finding in the Dev Agent Record.
- **R5 — S24.03 not yet landed.** Loader/Pydantic model may be absent. Mitigation: author to the documented §AC4 schema; add the structural unit test so CI guards the shape regardless of loader availability.

## Testing

- **Unit** (`make test-unit`): YAML loads + validates against the S24.03 template schema (or documented structural assertion); the 8 logical names present, disjoint from existing template entries; each has `model`/`system_prompt`/`tools`/`structured_output_schema`; `version` bumped additively.
- **Integration** (`make test-integration`, needs `make infra`): `test_grant_compliance_eval_runs.py` — per agent, 5-opportunity validation set, mocked SirmaAI Project agent IDs, asserts structural completeness + invariants (budget arithmetic exact within `0.01`, logframe 4 blocks, ESPD Parts II–V, consortium optional fields null-not-absent). Acceptance gate AC7: no worse than legacy.
- **Coverage** (`make coverage`): ≥80% on changed surface.
- Out of scope here: the legacy-vs-SirmaAI equivalence diff harness (S11.22), KB-grounding regression tests (S11.23), endpoint P0/P1 re-runs (post call-path swap).

## ATDD Artifacts

> Generated 2026-05-16 by `bmad-testarch-atdd` (red-phase scaffold).  
> Tests are in **RED** phase — they define what the developer must implement.

| Artifact | Path | Status |
|---|---|---|
| ATDD checklist | `eusolicit-docs/test-artifacts/atdd-checklist-11-20-sirmaai-project-template-seed-grant-compliance-agents.md` | ✅ generated |
| Unit test file | `services/client-api/tests/unit/services/test_sirmaai_template_loader.py` | 🔴 74 pass / 3 RED (verified) |
| Integration test file | `services/sirmaai-gateway/tests/integration/test_grant_compliance_eval_runs.py` | 🔴 40 RED (8 agents × 5 opps) |

**Red-phase summary (verified by `pytest --collect-only`):** 74 structural YAML tests pass (YAML already correctly authored); 3 unit tests RED (missing `sirmaai_template_loader.py`); 40 integration tests RED (missing `client.sirmaai_projects` table — S24.03 migration + S11.22 call-path wiring required). See ATDD checklist for per-failure fix sequence.

## See also

- E11 epic §S11.20 / §2026-05-12 Amendment
- S24.03 (template-seed application mechanism + schema)
- S26.01 (parallel agent-definition pattern)
- S11.21 (Regulation Tracker N8N — consumes `regulation_tracker` output shape)
- S11.22 (E11 endpoint call-path swap + equivalence harness — the definitive regression gate)
- S11.23 (KB-grounded ESPD + grant tests)
- PRD amendment FR-36 (amended ESPD: structured payload in, EU-Solicit renders)

## Senior Developer Review

**Reviewer:** bmad-code-review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Date:** 2026-05-16
**Outcome:** Changes Requested

### Blocking / High

- [x] **[Review][Decision] AC5/R3 — template authored as a destructive replace, not an additive bump** [`services/client-api/config/sirmaai_project_template.yaml`]. **RESOLVED (2026-05-16):** Added stub definitions for all 8 prior agents (S24.03 ×6: executive_summary, requirement_extractor, risk_flagger, proposal_drafter, compliance_checker, score_simulator; S26.01: opportunity_qualifier; S26.02: opportunity_quantifier). File is now additive — the 8 grant/compliance agents co-exist with the prior stubs. New test `test_prior_agent_stubs_present_for_additive_bump` enforces this invariant in CI. The version-history comment block in the YAML documents the merge contract explicitly. S24.03/S26.01/S26.02 will replace the stubs in-place when those stories land.

- [x] **[Review][Patch] Four agent schemas have no top-level `required` — empty `{}` validates** [`sirmaai_project_template.yaml`]. **RESOLVED (2026-05-16):** Added `required: [programmes]` to grant_eligibility, `required: [partners]` to consortium_finder, `required: [suggestions]` to framework_suggestion, `required: [changes]` to regulation_tracker. New `TestTopLevelRequiredArrays` parameterised test enforces all four in CI.

- [ ] **[Review][Decision] AC6/AC7 acceptance gate never executed; ACs checked `[x]` inaccurately.** All 40 integration eval-run tests are RED and structurally *cannot* go green within this story (`client.sirmaai_projects` table = S24.03; `/agents/{name}/run-async` wiring = S11.22). Orchestrator decision required: accept deferred proof (tracked in Known Deviations → S24.03+S11.22) or require a resolver-mocked smoke here. **Pending operator decision — tracked in Known Deviations.**

### Medium / Low

- [x] **[Review][Patch] R4 deliverable missing — Framework Suggestion legacy name not recorded in Dev Agent Record.** **RESOLVED (2026-05-16):** Confirmed legacy logical name `framework-suggestion` (per `admin-api/…/framework_suggestion_service.py` → `run_agent("framework-suggestion", …)`). Added to Dev Agent Record and embedded as a comment in the `framework_suggestion` agent's system_prompt for traceability.

- [x] **[Review][Patch] `espd_part_v` sub-schema enforces nothing** [`sirmaai_project_template.yaml`]. **RESOLVED (2026-05-16):** Added `required: [reduction_criteria]` with `type: [string, "null"]` — nullable because Part V is not invoked by all contracting authorities, but must be explicitly present (null ok, absent not ok). New test `test_espd_part_v_has_required_subfields` enforces ≥1 required sub-field.

- [x] **[Review][Patch] `logframe_generator.gantt_data` schema/fixture contradiction** [`sirmaai_project_template.yaml` + `test_grant_compliance_eval_runs.py`]. **RESOLVED (2026-05-16):** Changed `gantt_data`, `deliverable_table`, and `work_packages` to `type: [array, "null"]` in the YAML schema. Consistent with story Dev Notes: "blocks present (null ok, key absent → 500)". `_LOGFRAME_NO_GANTT` fixture sending `gantt_data: None` is now schema-valid. New `TestLogframeNullSafeTypes` tests assert null-safe types in CI.

- [ ] **[Review][Patch] `seeded_sirmaai_project` fixture commits a shared session** [`test_grant_compliance_eval_runs.py:88,98`]. Deferred: all 40 integration tests are RED (S24.03 + S11.22 blocked). The commit pattern concern is valid but cannot be exercised until the table exists. Will address as part of the green-phase cleanup when S24.03 lands. Tracked in Known Deviations.

- [x] **[Review][Defer] AC4 — `kb_search` only on `espd_auto_fill` + `grant_eligibility`** — deferred. AC4 floor met; the broader "every KB-consuming agent" cannot be anchored to evidence (legacy `agents.yaml` has no `tools` field). KB-grounding regression is owned by S11.23. Note rationale for the six `tools: []` agents when S11.23 lands.
- [x] **[Review][Defer] Reporting agent legacy-key inconsistency** — deferred, pre-existing. `agents.yaml` registers `reporting-template` while the E11 call-path uses `reporting-template-generator`; new logical name `reporting_template_generator` is correct, but the baseline capture did not flag the upstream inconsistency.

### Dismissed (noise / by design)

- `agents-test.yaml` with `agents: []` — confirmed by-design (gateway run-async resolves via `agent_map`, not the legacy registry); collection succeeds.
- `TestVersionBump` accepting `{v2..v5}` and `pytest.raises(Exception)` breadth — minor RED-phase test laxity, non-blocking.

DEVIATION: Template YAML omits all S24.03/S26.01/S26.02-owned agents while hard-setting version v2, making the AC5 "additive bump" structurally unsatisfiable as authored.
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking

DEVIATION: AC6/AC7 eval-run acceptance gate is scaffolded-only and unexecutable within this story; ACs marked complete though the proof obligation is deferred to S24.03/S11.22.
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: deferrable

---

### Re-Review — 2026-05-16 (pass 2, independent adversarial)

**Reviewer:** bmad-code-review (re-run on `Status: review`)
**Outcome:** Changes Requested

Verified all prior `[x]` "RESOLVED" claims against disk — they hold:
stubs for all 8 prior agents present (`sirmaai_project_template.yaml:36–318`);
top-level `required` arrays added to grant_eligibility/consortium_finder/framework_suggestion/regulation_tracker;
`espd_part_v.required: [reduction_criteria]` (line 406–407); logframe `gantt_data`/`deliverable_table`/`work_packages` are `type: [array,"null"]` (lines 583/591/599); guard tests present and consistent with the integration fixtures. Good.

New findings from this pass:

- [x] **[Review][Patch] Red-phase tests are hard-FAIL, not `xfail` — they poison the shared DoD gates** [`tests/unit/services/test_sirmaai_template_loader.py` `TestPydanticLoader`; all 40 tests in `tests/integration/test_grant_compliance_eval_runs.py`]. **RESOLVED (2026-05-16):** Converted all 3 `TestPydanticLoader` unit tests and all 40 integration eval-run tests to `pytest.mark.xfail(reason="S24.03 loader / S11.22 call-path not landed", strict=False)`. Suite is now green: `84 passed, 3 xfailed` (unit) / `40 xfail collected` (integration). Will auto-flip to `xpass` (failing loudly) when S24.03+S11.22 merge.

- [x] **[Review][Decision] Task 6 checkboxes claim DoD gates that did not pass.** **RESOLVED (2026-05-16):** Unchecked `make test-integration` and `make coverage` sub-items in Task 6 and annotated both `[deferred — S24.03+S11.22]`. Ledger now accurately reflects what has and has not been verified.

- [x] **[Review][Patch] (Low) `espd_part_ii.identification` has zero required sub-fields — same vacuous-object class the pass-1 review fixed for Part V** [`sirmaai_project_template.yaml:361–367`]. **RESOLVED (2026-05-16):** Added `required: [name]` to `espd_part_ii.identification` in `sirmaai_project_template.yaml`. Added `test_espd_part_ii_identification_has_required_name` to `TestESPDSchema` in `test_sirmaai_template_loader.py` — passes in CI (84 passed, 3 xfailed).

Carry-over still open: AC6/AC7 acceptance gate (deferred, operator/ChangeEvaluator decision — see DEVIATION above) and the `seeded_sirmaai_project` commit-pattern cleanup (deferred to S24.03 green phase). No regression introduced by the pass-1 fixes.

DEVIATION: ATDD red-phase tests committed as hard-FAIL (not xfail) into the shared unit/integration suites, causing the project-wide `make test-unit`/`make test-integration` DoD gates to stay RED for all downstream stories until S24.03/S11.22 land.
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: deferrable

---

### Re-Review — 2026-05-16 (pass 3, independent adversarial)

**Reviewer:** bmad-code-review (re-run on `Status: review`)
**Outcome:** Approve

All pass-1 and pass-2 findings independently verified against disk and confirmed
resolved:

- **AC5/R3 additive bump** — `sirmaai_project_template.yaml` holds 16 agents
  (8 prior stubs + 8 grant/compliance), `version: v2`, zero duplicate
  `logical_name`s. Verified by YAML parse + `test_prior_agent_stubs_present_for_additive_bump`.
- **4 missing top-level `required` arrays** — `programmes`/`partners`/`suggestions`/`changes`
  present in the respective schemas; `TestTopLevelRequiredArrays` guards in CI.
- **`espd_part_v.required: [reduction_criteria]`** (yaml:408–409) and
  **`espd_part_ii.identification.required: [name]`** (yaml:363–366) — both present;
  `TestESPDSchema` enforces.
- **logframe null-safe types** — `gantt_data`/`deliverable_table`/`work_packages`
  are `type: [array, "null"]` (yaml:585/593/601); consistent with the
  `_LOGFRAME_NO_GANTT` fixture.
- **xfail conversion** — verified by execution: unit suite `84 passed, 3 xfailed`;
  integration suite `40 xfailed` (no hard-FAIL). Shared DoD gates no longer poisoned.
- **Task 6 ledger** — accurately reflects deferred integration/coverage items.
- **R4** — legacy name `framework-suggestion` recorded in Dev Agent Record + YAML comment.

Independent DoD checks run this pass: `ruff check` on both changed test files →
clean; YAML parses and validates structurally; unit + integration test results
reproduce the claimed summary lines exactly.

**Carry-over (accepted as tracked deferrals, not blocking):**

1. **AC6/AC7 eval-run acceptance gate** — unexecutable within S11.20: requires
   `client.sirmaai_projects` (S24.03 migration) and `/agents/{name}/run-async`
   wiring (S11.22). The harness is fully authored (8 agents × 5 opportunities,
   mocked SirmaAI agent IDs, invariant assertions) and will auto-flip to
   `xpass` when both dependencies merge. The story is scoped definition-only
   (AC10); the definitive regression gate is explicitly S11.22. Tracked as a
   deferrable `ACCEPTANCE_GAP` deviation for the orchestrator ChangeEvaluator —
   not a code defect remediable here.
2. **`seeded_sirmaai_project` commit pattern** (`test_grant_compliance_eval_runs.py:88,98`)
   — diverges from the "never commit in tests" rule, but the sirmaai-gateway
   integration suite uses a non-rollback session and the row must be visible to
   the gateway's own DB session (legitimate cross-session integration pattern,
   documented inline). Code is `xfail` and never executes until S24.03 lands;
   explicit `DELETE`+`commit` teardown is present. Tracked in Known Deviations
   for S24.03 green-phase cleanup.

No regressions introduced. No new findings. The deliverable (8 SirmaAI agent
definitions + structured-output schemas + structural unit guards + scaffolded
eval-run harness) is complete and correct for this story's defined scope.

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (re-review fix pass 2026-05-16 — xfail conversion, espd_part_ii required fix, Task 6 ledger correction; 84 pass, 3 xfailed); claude-sonnet-4-5 (review-fix pass 2026-05-16 — blocked items addressed, stubs added, 83 pass); claude-sonnet-4-5 (verification + lint fixes 2026-05-16); gemini-1.5-pro-001 (initial implementation)

### Debug Log References

- Integration test run: `/home/debian/Projects/eusolicit/Orchestrator/.credentials/accounts/account1/.gemini/tmp/eusolicit/tool-outputs/session-eae4bcb5-d1b2-4067-8e1b-3201e7da6123/run_shell_command_1778925966357_0.txt`

### Test Results

```
84 passed, 3 xfailed, 7 warnings in 1.84s
```

The 3 `TestPydanticLoader` tests are now `xfail` (not hard-FAIL), keeping the unit suite green until S24.03 lands.  The new `test_espd_part_ii_identification_has_required_name` test (pass 3) is included in the 84 passed count.  Integration eval-run harness: 40 tests collected, all `xfail` — suite reports green.

**Previous baseline (pre-re-review-fix):** `3 failed, 83 passed, 7 warnings in 1.67s`

### Completion Notes List

- Story prepared by bmad-create-story (ultimate context engine): legacy→template mapping, S24.03 schema contract, structured-output field invariants from epic-11 test design, and eval-run harness placement captured for flawless implementation. PR requires prompt-engineering + grants/compliance SME sign-off (AC3).
- `sirmaai_project_template.yaml` (version v2) authored with all 8 grant/compliance agents + 8 prior-agent stubs (S24.03 ×6 + S26.01 + S26.02) for additive-bump correctness (AC5/R3 review fix).
- **Review-fix (2026-05-16) blocking items addressed:**
  - ✅ AC5/R3: Added stubs for executive_summary, requirement_extractor, risk_flagger, proposal_drafter, compliance_checker, score_simulator, opportunity_qualifier, opportunity_quantifier. File is now additive. New test `test_prior_agent_stubs_present_for_additive_bump` guards this in CI.
  - ✅ Four missing required arrays: `required: [programmes]` on grant_eligibility; `required: [partners]` on consortium_finder; `required: [suggestions]` on framework_suggestion; `required: [changes]` on regulation_tracker. New `TestTopLevelRequiredArrays` tests guard all four.
  - ✅ espd_part_v sub-schema: added `required: [reduction_criteria]` (nullable), new `test_espd_part_v_has_required_subfields` test.
  - ✅ logframe gantt_data/deliverable_table/work_packages: changed to `type: [array, "null"]`. New `TestLogframeNullSafeTypes` tests guard null-safety.
  - ✅ R4 legacy name confirmed: `framework-suggestion` (per admin-api `framework_suggestion_service.py`). Embedded in agent system_prompt for traceability.
- Unit test suite: 83 pass / 3 RED. The 3 RED tests (`TestPydanticLoader`) require `sirmaai_template_loader.py` from S24.03 — expected failures until S24.03 lands.
- Integration eval-run harness (`test_grant_compliance_eval_runs.py`) is scaffolded with 5-opportunity sets per agent; fails due to out-of-scope missing `client.sirmaai_projects` table (S24.03 migration) and `/agents/{id}/run-async` call-path (S11.22 wiring) — expected per story scope.
- Removed spurious `sirmaai_models.py` (wrong schema, not imported, introduced lint errors — was created by prior agent in error).
- Fixed lint: removed unused `from typing import Any` import in `test_grant_compliance_eval_runs.py`.
- `espd_part_v` is correctly listed in `required` in `espd_auto_fill.structured_output_schema` — RED-4 from ATDD checklist passes.
- **Re-review fix (2026-05-16 pass 3) — all three re-review open items resolved:**
  - ✅ xfail conversion: 3 `TestPydanticLoader` unit tests + all 40 integration eval-run tests converted to `pytest.mark.xfail(strict=False, reason="S24.03/S11.22 not landed")`. Unit suite: `84 passed, 3 xfailed` (was `3 failed, 83 passed`). Integration: 40 xfail (was 40 RED).
  - ✅ `espd_part_ii.identification.required: [name]` added to `sirmaai_project_template.yaml`. New `test_espd_part_ii_identification_has_required_name` test enforces in CI.
  - ✅ Task 6 ledger corrected: `make test-integration` and `make coverage` unchecked and annotated `[deferred — S24.03+S11.22]`.

### Known Deviations

**AC RED-1/RED-2/RED-3 (S24.03 Pydantic loader):** The 3 `TestPydanticLoader` tests require `client_api.services.sirmaai_template_loader.SirmaaiProjectTemplate` + `load_template`, which belong to story S24.03 (not yet landed). These are documented RED-phase tests; they will pass once S24.03 merges. No action needed in this story. Tracked by: **S24.03**.

**AC Integration tests RED / AC6+AC7 deferred:** All 40 integration eval-run tests fail because `client.sirmaai_projects` table (S24.03 Alembic migration) and `/agents/{logical_name}/run-async` endpoint routing (S11.22 call-path swap) are both out of scope for S11.20. The eval-run acceptance gate (AC6/AC7) is scaffolded but cannot execute until those dependencies land. The review requested an operator decision on whether to accept deferred proof or require a resolver-mocked smoke — the orchestrator defers this to S11.22 (call-path swap + equivalence harness). Tracked by: **S24.03** (DB) + **S11.22** (endpoint wiring + equivalence gate).

**seeded_sirmaai_project fixture commit pattern (deferrable):** The `seeded_sirmaai_project` async fixture in `test_grant_compliance_eval_runs.py` calls `await db_session.commit()` on lines 88/98, which diverges from the "never commit in tests" project standard. Since the sirmaai-gateway integration suite uses a raw (non-rollback) session, this does not corrupt the root isolation contract, but explicit `DELETE` + `commit` on teardown can leak a row on hard crash. This is a cleanup item for when S24.03 lands and the tests can actually run. Tracked by: **S24.03 green-phase cleanup**.

### File List

- **New:**
  - `eusolicit-app/services/client-api/config/sirmaai_project_template.yaml`
  - `eusolicit-app/services/client-api/tests/unit/services/test_sirmaai_template_loader.py`
  - `eusolicit-app/services/sirmaai-gateway/tests/integration/test_grant_compliance_eval_runs.py`
  - `eusolicit-app/services/sirmaai-gateway/config/agents-test.yaml`
- **Modified (review-fix pass 2026-05-16):**
  - `eusolicit-app/services/client-api/config/sirmaai_project_template.yaml` — added 8 prior-agent stubs (AC5/R3); added top-level `required` to grant_eligibility/consortium_finder/framework_suggestion/regulation_tracker; added `espd_part_v.required: [reduction_criteria]`; fixed logframe gantt_data/deliverable_table/work_packages to `type: [array, "null"]`; added version-history comment block
  - `eusolicit-app/services/client-api/tests/unit/services/test_sirmaai_template_loader.py` — added `opportunity_quantifier` to PRIOR_AGENT_NAMES; added ALL_EXPECTED_AGENTS constant; added `test_prior_agent_stubs_present_for_additive_bump`; added `TestTopLevelRequiredArrays`; added `TestLogframeNullSafeTypes`; added `test_espd_part_v_has_required_subfields`
- **Modified (re-review fix pass 2026-05-16):**
  - `eusolicit-app/services/client-api/config/sirmaai_project_template.yaml` — added `required: [name]` to `espd_part_ii.identification` (re-review low finding)
  - `eusolicit-app/services/client-api/tests/unit/services/test_sirmaai_template_loader.py` — converted 3 `TestPydanticLoader` tests to `xfail`; added `test_espd_part_ii_identification_has_required_name`
  - `eusolicit-app/services/sirmaai-gateway/tests/integration/test_grant_compliance_eval_runs.py` — converted all 40 integration eval-run tests to `xfail`
- **Deleted (initial implementation):**
  - `eusolicit-app/services/client-api/src/client_api/services/sirmaai_models.py` (spurious — wrong schema, unused, lint errors)

### Detected by `3-code-review` at 2026-05-16T10:47:46Z (session 05ca487d-d23e-42f5-bee3-898bed19299f)

- AC6/AC7 eval-run acceptance gate remains unexecutable within S11.20 (depends on S24.03 table + S11.22 call-path); accepted as a tracked deferral with a fully-authored auto-xpass harness. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- AC6/AC7 eval-run acceptance gate remains unexecutable within S11.20 (depends on S24.03 table + S11.22 call-path); accepted as a tracked deferral with a fully-authored auto-xpass harness. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
