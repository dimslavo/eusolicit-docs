---
workflowStatus: 'completed'
totalSteps: 5
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
nextStep: ''
lastSaved: '2026-05-25'
workflowType: 'testarch-test-design'
designLevel: 'epic'
epicNum: 12
inputDocuments:
  - '/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epics/epic-12-admin-platform.md'
  - '/home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/test-design-architecture.md'
  - '/home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/test-design-qa.md'
  - '/home/debian/Projects/eusolicit/eusolicit-docs/project-context.md'
  - '.claude/skills/bmad-testarch-test-design/resources/knowledge/risk-governance.md'
  - '.claude/skills/bmad-testarch-test-design/resources/knowledge/probability-impact.md'
  - '.claude/skills/bmad-testarch-test-design/resources/knowledge/test-levels-framework.md'
  - '.claude/skills/bmad-testarch-test-design/resources/knowledge/test-priorities-matrix.md'
outputFile: 'eusolicit-docs/test-artifacts/test-design-epic-12.md'
---

# Test Design Progress — Epic 12 (Admin Platform)

## Step 1: Detect Mode & Prerequisites

**Mode:** Epic-Level (Phase 4). User explicitly requested an epic-level test design for Epic 12; epic file + ACs present. Prerequisites met (epic ACs + system-level context). Run on 2026-05-25.

## Step 2: Load Context

- Config: `_bmad/tea/config.yaml` (test_stack_type=fullstack, playwright+pactjs utils enabled, test_artifacts=`eusolicit-docs/test-artifacts`).
- Loaded Epic 12 spec (2 stories: 12.1 Tenant Management, 12.2 KPI Dashboards).
- Loaded system-level `test-design-architecture.md` + `test-design-qa.md` for context (inherits R-002 cross-tenant isolation, two-tier RBAC contract, R-014 HMAC discipline).
- Loaded knowledge fragments: risk-governance, probability-impact, test-levels-framework, test-priorities-matrix.
- **Existing coverage verified in code:** `admin-api` present with `tests/services/test_tenant_service.py`, `tests/middleware/test_ip_allowlist.py`, `tests/integration/test_pricing_tiers_admin_only.py`, `test_stage_mapping_admin.py`. Admin Next.js app (:3001) with `[locale]` routes. Grounds R-001 (allowlist) and R-002 (tenant mutation).

## Step 3: Risk Assessment

8 risks (3 high ≥6). Top categories: SEC (admin-portal access + privilege), DATA (tenant state integrity + KPI freshness). High: R-001 IP-allowlist bypass (SEC), R-002 incorrect/cross-tenant tenant mutation (DATA/SEC), R-003 stale/incorrect KPI dashboard via MV refresh failure (DATA). Medium: R-004 audit completeness, R-005 dashboard perf, R-006 MV bootstrap/migration drift. Low: R-007 non-operator authz, R-008 Recharts breakage.

## Step 4: Coverage Plan

39 scenarios — P0=12, P1=14, P2=10, P3=3. Levels: API/Middleware (allowlist, authz, audit, perf), Integration (MV refresh task, MV bootstrap), E2E (operator journey, UI↔DB data-assertion, render states, visual regression). Execution: smoke → P0 → P1 in PR; P2/P3 + E2E nightly. Effort ~36–58h (~5–8 days). Gates: P0 100%, P1 ≥95%, SEC 100%, admin-api coverage ≥80%.

## Step 5: Generate Output & Validation

**Mode:** sequential (single epic-level artifact).
**Output:** `eusolicit-docs/test-artifacts/test-design-epic-12.md` (regenerated 2026-05-25 from `test-design-template.md`).
**Validation:** epic-level checklist satisfied — unique risk IDs, P×I 1–3 scored, high-priority flagged, mitigation plans + owners + timelines, coverage matrix with levels/priority/risk-link, execution order, interval-based estimates, quality gates, not-in-scope, entry/exit criteria, interworking & regression. No browser sessions opened (doc/code analysis only); artifacts under test_artifacts. on_complete hook empty — skipped.
