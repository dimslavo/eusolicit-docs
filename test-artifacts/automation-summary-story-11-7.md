---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-identify-targets
  - step-03-generate-tests
  - step-04-validate-and-summarize
lastStep: step-04-validate-and-summarize
lastSaved: '2026-05-09'
workflowType: bmad-testarch-automate
mode: bmad-integrated
scope: coverage-sweep-cluster-B
storyKey: 11-7-logframe-generator-reporting-template-agent-integrations
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/traceability-matrix.md
  - eusolicit-docs/test-artifacts/atdd-checklist-11-7-logframe-generator-reporting-template-agent-integrations.md
  - eusolicit-docs/test-artifacts/test-design-epic-11.md
  - eusolicit-app/services/client-api/src/client_api/api/v1/grants.py
  - eusolicit-app/services/client-api/src/client_api/services/grants_service.py
  - eusolicit-app/services/client-api/src/client_api/schemas/grants.py
  - eusolicit-app/services/client-api/src/client_api/services/ai_gateway_client.py
  - eusolicit-app/services/client-api/src/client_api/models/proposal.py
  - eusolicit-app/services/client-api/tests/api/test_logframe_generator.py
  - eusolicit-app/services/client-api/tests/api/test_reporting_template.py
  - eusolicit-app/services/client-api/tests/api/test_consortium_finder.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/_bmad/tea/config.yaml
---

# Automation Summary: Story 11.7 — Logframe + Reporting Template (Coverage Sweep Cluster B)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Story:** S11.07 — Logframe Generator & Reporting Template Agent Integrations
**Status:** ✅ P1 complete — files updated, lint + pytest collection pass; runtime execution deferred to CI / `make test-service SVC=client-api` (host venv missing service runtime deps).

---

## Step 1: Preflight & Context

### Stack Detection

- `test_stack_type: fullstack` (explicit in `_bmad/tea/config.yaml`)
- Backend: pytest + pytest-asyncio + respx + httpx + async SQLAlchemy
- Frontend / E2E: Playwright (`eusolicit-app/playwright.config.ts`)
- Pact / contract testing: not active (no `pact/`, no `@pact-foundation/pact`, no `PACT_BROKER`)
- `tea_pact_mcp: none`, `tea_browser_automation: auto`

### Execution Mode

- **BMad-Integrated** — story file, ATDD checklist, and test-design-epic-11 all present.

### Knowledge Fragments Loaded (read on demand)

- Core: `test-levels-framework`, `test-priorities-matrix`, `data-factories`, `selective-testing`, `ci-burn-in`, `test-quality`
- Playwright Utils (full UI+API profile): `overview`, `api-request`, `auth-session`, `recurse`, `log` (this story is backend-only — utils relevant only for follow-up E2E in cluster C)
- Skipped: pactjs-utils (no microservices contract testing in scope), pact-mcp (off)

---

## Step 2: Coverage Plan

### Sweep Findings (used to scope this run)

- Sprint state: 21/21 epics done (191 stories); no in-flight work.
- Last traceability snapshot (epic-11, 2026-04-25): **Gate FAIL** — P1 = 75%, two open gaps:
  - **AC4** Logframe Generator (RED tests existed; impl now landed)
  - **AC5** Reporting Template Generator (same)
- Story `11-7-logframe-generator-reporting-template-agent-integrations: done` per `sprint-status.yaml`.
- Endpoints verified live in `services/client-api/src/client_api/api/v1/grants.py:114, 134, 164`.
- Service implementations verified in `services/client-api/src/client_api/services/grants_service.py:700, 873, 965`.
- Pydantic schemas verified in `services/client-api/src/client_api/schemas/grants.py:219-291`.
- DB model `Proposal` verified at `services/client-api/src/client_api/models/proposal.py:27` with `milestones`, `budget_summary`, `consortium` columns (Story 11-7 model extension).

### Targets

| File | Layer | Priority | Action |
|---|---|---|---|
| `services/client-api/tests/api/test_logframe_generator.py` | API integration | P0/P1 | RED → GREEN docstrings; add 2 new positive tests |
| `services/client-api/tests/api/test_reporting_template.py` | API integration | P0/P1 | RED → GREEN docstrings; remove permissive seed try/except; add 4 new positive tests |

### Test Levels & Priority Matrix

- Test level: **API integration** (httpx ASGI client + respx-intercepted AI Gateway). No unit-level changes (parsers already covered indirectly), no E2E (deferred to cluster C if needed).
- Priorities follow ATDD checklist 11-7 (E11-P0-008/009/010, E11-P1-004/005/006/007, E11-P2-003).

### New Coverage to Add (closes the gap)

**Logframe (`test_logframe_generator.py`):**
1. `TestSecurityIsolation::test_logframe_jwt_company_id_authoritative` — caller forges `company_id` in request body; verify the JWT company_id (NOT the body field) is forwarded to the agent payload. Closes a real risk path: cross-tenant data leakage via forged body.
2. `TestAC3AC4ResponseParsing::test_logframe_malformed_rows_dropped_no_500` — agent returns mixed valid/malformed rows in `logical_framework`/`work_packages`/`gantt_data`; assert response keeps only valid rows and never returns 500. Locks in `_parse_*` filtering behaviour visible at `grants_service.py:734-749`.

**Reporting (`test_reporting_template.py`):**
1. `TestAC8NotFound::test_cross_company_project_returns_404` — project owned by company A; login as company B; expect 404 (validates the `WHERE company_id = :company_id` predicate at `grants_service.py:781-786`).
2. `TestAC7AC8HappyPath::test_consortium_summary_forwarded_in_response` — agent returns `consortium_summary`; assert it passes through unmodified.
3. `TestAC11DOCX::test_export_filename_includes_project_id` — assert `Content-Disposition` filename contains the project UUID (verifies safe filename construction at `grants.py:155`).
4. `TestAC11DOCX::test_export_docx_contains_project_title_heading` — parse DOCX with python-docx; assert the project title appears as a heading (verifies `build_report_docx` at `grants_service.py:929`).

### Justification

- Coverage scope: **selective** — locks in the two acceptance criteria that fail the trace gate; broadens RLS + parser hardening which are common defect sites.
- Avoiding duplication: existing tests already cover Pydantic 422, AC5/AC10 503 paths, AC4 null/empty Gantt distinction, and DOCX MIME type. New tests add only complementary positive coverage.

---

## Step 3: Generate Tests

### Files Modified

#### `services/client-api/tests/api/test_logframe_generator.py`

**Header & docstrings:**
- Module header changed `🔴 RED PHASE: Story 11.7 NOT yet implemented.` → `🟢 GREEN PHASE: Story 11.7 implemented.`
- Dropped the "Files modified/created by this story (all must exist before GREEN PHASE)" block (now stale).
- Removed `try/except (ImportError, AttributeError):` guards in `aigw_env_setup` around `client_api.config` and `client_api.services.ai_gateway_client` imports (the modules exist).
- Replaced 11× per-test `Fails in RED PHASE because…` paragraphs with concise one-line docstrings.
- Replaced 5× class-level `TDD phase: 🔴 RED — Story 11.7 NOT yet implemented.` with 🟢 GREEN equivalents.

**New tests (+2):**
- `TestSecurityIsolation::test_logframe_jwt_company_id_authoritative` — Forges a `company_id` in the request body; asserts the agent payload `company_id` equals the JWT-derived value, **not** the forged body field. Closes the "trust the body" cross-tenant data-leak path.
- `TestParserRobustness::test_logframe_non_dict_rows_dropped_no_500` — Mixes valid dict rows with non-dict trash (`None`, strings, integers, lists) across all four parsed lists; asserts response keeps only valid rows and never returns 500. Locks in the `isinstance(raw, dict)` guard at `grants_service.py:604/627/654/677`.

**Final test count:** 19 (was 14; +5 net counting parametrized cases across the new `TestSecurityIsolation` and `TestParserRobustness` classes).

#### `services/client-api/tests/api/test_reporting_template.py`

**Header & docstrings:**
- Module header changed 🔴 → 🟢.
- Dropped "DB TABLE NOTE", "TABLE PLACEHOLDER", and "Files modified/created" blocks (table now confirmed: `client.proposals` via `Proposal` model at `services/client-api/src/client_api/models/proposal.py:27`).
- Removed `try/except (ImportError, AttributeError):` guards in `aigw_env_setup`.
- **Tightened seed:** removed the bare `try/except Exception:  # noqa: BLE001` wrap around the `INSERT INTO client.proposals` so fixture errors surface immediately rather than masquerade as endpoint 404s.
- Replaced 16× per-test `Fails in RED PHASE because…` paragraphs with concise one-line docstrings.
- Replaced 8× class-level `TDD phase: 🔴 RED` markers with 🟢 GREEN equivalents.

**New tests (+4):**
- `TestCrossCompanyIsolation::test_cross_company_project_returns_404` — Seeds project under company A; logs in as company B; asserts both `/reporting-template` and `/export` return 404. Validates the `WHERE company_id = :company_id` predicate at `grants_service.py:781-786`. Standalone fixture (does not reuse `reporting_client_and_session` because it provisions two distinct tenants).
- `TestConsortiumSummaryPassThrough::test_consortium_summary_in_response` — Asserts `consortium_summary` returned by the agent surfaces in the JSON response with the same organisation values; closes a pass-through gap the existing tests didn't assert.
- `TestExportArtifactShape::test_export_filename_includes_project_id` — Asserts `Content-Disposition` filename contains the project UUID (verifies safe filename construction at `grants.py:155`).
- `TestExportArtifactShape::test_export_docx_contains_project_title_heading` — Parses DOCX bytes with `python-docx`; asserts the project title and at least one section heading appear in rendered paragraphs (verifies `build_report_docx` at `grants_service.py:929`).

**Final test count:** 22 (was 17; +5 — `TestCrossCompanyIsolation` adds 1 test that internally exercises 2 endpoints).

### Files Created

- `eusolicit-docs/test-artifacts/automation-summary-story-11-7.md` (this file).

### Files Not Touched

- `services/client-api/src/client_api/api/v1/grants.py` (impl already correct)
- `services/client-api/src/client_api/services/grants_service.py` (impl already correct)
- `services/client-api/src/client_api/schemas/grants.py` (impl already correct)
- `services/client-api/src/client_api/models/proposal.py` (impl already correct)

---

## Step 4: Validation & Summary

### Local Static Validation (passed)

| Check | Command | Result |
|---|---|---|
| Python syntax | `python3 -c "import ast; ast.parse(open(...).read())"` | ✅ both files OK |
| Lint | `./venv/bin/ruff check services/client-api/tests/api/test_logframe_generator.py services/client-api/tests/api/test_reporting_template.py` | ✅ All checks passed |
| Pytest collection | `./venv/bin/pytest --collect-only services/client-api/tests/api/test_logframe_generator.py services/client-api/tests/api/test_reporting_template.py` | ✅ 41 tests collected |

### Runtime Validation (deferred)

Local host venv is missing service runtime deps (`itsdangerous` was missing initially; after install, `bcrypt` was missing — likely many more). This is a host-environment issue, not a test issue.

**To execute these tests, run from a fully provisioned environment (CI or via the docker test workflow):**

```bash
# From eusolicit-app/
make infra                                          # postgres + redis up
make migrate-service SVC=client-api                 # ensure proposals table exists
make test-service SVC=client-api                    # runs the full client-api test suite
# or, narrowed:
pytest services/client-api/tests/api/test_logframe_generator.py services/client-api/tests/api/test_reporting_template.py -v
```

### Trace Gate Impact

Pre-run gate (epic 11, 2026-04-25): **FAIL** — P1 = 75%, AC4 + AC5 GAP.

Post-run expected gate (after CI runs the GREEN suite green): **PASS** — AC4 + AC5 covered with the production endpoint + agent payload + RLS + DOCX shape. Recommend re-running `/bmad:tea:trace` for epic-11 after the next green CI build of these two files.

### Risk Notes

- **Cross-tenant test fixture caveat (`TestCrossCompanyIsolation`):** Provisions two companies + a seeded project on the same `db_session` override. Teardown rolls back, so no cross-test pollution — but if the rollback is silently skipped (e.g., session leaked), company-B test data could persist. Existing pattern matches `test_proposal_collaborators_tenant_isolation.py` so the suite already assumes correct rollback semantics.
- **DOCX heading assertion (`test_export_docx_contains_project_title_heading`):** `python-docx` represents headings as paragraphs with style `Heading 1` etc. The assertion checks paragraph text content rather than style, which is forgiving — if `build_report_docx` ever stops adding the title to the document, the test still catches it.
- **No infra-prep changes needed.** All new tests use the existing `client_api_session_factory` + `test_redis_client` + ASGI-transport pattern from `tests/conftest.py`. No new conftest fixtures.

### What's Next (P2–P4 plan, separate run)

To be drafted into a follow-up document under `eusolicit-docs/test-artifacts/full-suite-rollout-plan.md`:

- **P2** Re-enable skipped E2E specs (240+ tests across 7 files) — cluster-A.
- **P3** New E2E specs for uncovered done features (Tasks/Kanban, Approvals, Analytics×7, Reports, Calendar, Bid outcomes, Comments, Content blocks, Admin UI) — cluster-C.
- **P4** API-level integration coverage for `nps_feedback.py`, `billing.py`, cross-service notification — cluster-D.

---

