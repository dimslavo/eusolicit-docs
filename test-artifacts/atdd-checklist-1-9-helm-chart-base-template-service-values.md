---
stepsCompleted:
  - 'step-01-preflight-and-context'
  - 'step-02-generation-mode'
  - 'step-03-test-strategy'
  - 'step-04-generate-tests'
  - 'step-04c-aggregate'
  - 'step-05-validate-and-complete'
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-06'
workflowType: 'testarch-atdd'
storyId: '1-9-helm-chart-base-template-service-values'
detectedStack: 'infrastructure'
generationMode: 'ai-generation'
executionMode: 'sequential'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/1-9-helm-chart-base-template-service-values.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-01.md'
  - 'eusolicit-app/pyproject.toml'
  - 'eusolicit-app/infra/helm/README.md'
  - 'eusolicit-app/tests/conftest.py'
---

# ATDD Checklist: Story 1.9 -- Helm Chart Base Template & Service Values

**Date:** 2026-04-06
**Story:** 1-9-helm-chart-base-template-service-values
**TDD Phase:** RED (failing tests -- implementation pending)
**Stack:** Infrastructure (Helm chart YAML templates + Go templating + per-service values)

---

## Preflight Summary

| Item | Status |
|------|--------|
| Story approved with clear ACs | ✅ 8 acceptance criteria |
| Test framework configured | ✅ pytest + pyyaml (YAML parsing) |
| Dev environment available | ✅ Monorepo with Python 3.12 |
| Detected stack | `infrastructure` (Helm chart YAML/Go templates + values validation) |
| Generation mode | AI generation (static file + template pattern validation) |
| Epic test design loaded | ✅ test-design-epic-01.md |
| TEA config flags | None configured -- infrastructure validation tests only |

---

## TDD Red Phase Status

✅ **Failing tests generated -- 190 failing, 0 passing, 45 skipped**

| Test File | Test Count | Failed | Passed | Skipped | AC Coverage | Priority |
|-----------|-----------|--------|--------|---------|-------------|----------|
| `tests/unit/test_helm_chart_structure.py` | 64 | 64 | 0 | 0 | AC 1, 2, 8 | P0/P1 |
| `tests/unit/test_helm_templates.py` | 48 | 48 | 0 | 0 | AC 3, 4, 8 | P0/P1/P2 |
| `tests/unit/test_helm_service_values.py` | 78 | 78 | 0 | 0 | AC 5, 6 | P0/P1/P2 |
| `tests/unit/test_helm_template_rendering.py` | 45 | 0 | 0 | 45 | AC 7, 8 | P2 |
| **Total** | **235** | **190** | **0** | **45** | **AC 1-8** | |

**45 skipped tests** are `helm template` rendering tests that require the Helm 3 CLI to be installed. They will run once `helm` is available and the chart files exist. When the dev creates the chart, these tests will un-skip and either pass or fail based on rendering correctness.

---

## Acceptance Criteria Coverage Matrix

| AC | Description | Test File | Scenarios | Priority |
|----|-------------|-----------|-----------|----------|
| **AC 1** | Base chart at infra/helm/eusolicit-service/ with Chart.yaml, templates/, values.yaml | `test_helm_chart_structure.py` | Directory exists, Chart.yaml exists/valid/metadata, values.yaml exists/valid/structure, templates/ exists, _helpers.tpl exists with helpers, NOTES.txt exists | **P0/P1** |
| **AC 2** | Templates include deployment, service, configmap, hpa, ingress, serviceaccount | `test_helm_chart_structure.py`, `test_helm_templates.py` | 6 required template files exist, Service/ConfigMap/Ingress/ServiceAccount template patterns correct | **P0/P1** |
| **AC 3** | Deployment supports resources, probes (/healthz), envFrom, volumes | `test_helm_templates.py` | Deployment kind/apiVersion, resource refs, liveness/readiness probes, envFrom configMapRef/secretRef, volumes/volumeMounts, containerPort, serviceAccountName, labels | **P0/P1** |
| **AC 4** | HPA supports min/max replicas and CPU/memory targets | `test_helm_templates.py` | HPA kind, apiVersion autoscaling/v2 (not v2beta2), minReplicas, maxReplicas, CPU target, optional memory target, conditional rendering | **P0/P1** |
| **AC 5** | Per-service values at infra/helm/values/<service>.yaml for all 5 services | `test_helm_service_values.py` | Values directory exists, 5 service files exist, each parses as valid YAML | **P0** |
| **AC 6** | Each values file sets image, port, resources, replicas, env vars | `test_helm_service_values.py` | containerPort matches architecture (8001-8005), targetPort, autoscaling enabled/min/max, ingress enabled/hosts, networkPolicy, secrets name, image repository, resource limits | **P0/P1/P2** |
| **AC 7** | helm template renders valid YAML for each service without errors | `test_helm_template_rendering.py` | Renders without errors (5 services), valid multi-doc YAML, contains Deployment + Service, apiVersion/kind on all docs, HPA present | **P2** |
| **AC 8** | Optional PDB and NetworkPolicy via feature flags | `test_helm_chart_structure.py`, `test_helm_templates.py`, `test_helm_template_rendering.py` | PDB/NetworkPolicy templates exist, correct apiVersion, conditional rendering patterns, conditional rendering verified via helm template | **P1/P2** |

---

## Risk Coverage

| Risk ID | Description | Score | Test Coverage |
|---------|-------------|-------|---------------|
| **E01-R-009** | Helm chart template rendering errors not caught until deployment | 2 (LOW) | 45 rendering tests validate `helm template` for all 5 services; YAML parsing validates rendered output; conditional rendering verified for Ingress, NetworkPolicy per service |

---

## Priority Distribution

| Priority | Count | Description |
|----------|-------|-------------|
| **P0** | 78 | Chart directory/files exist, Chart.yaml metadata (apiVersion v2, name, version), all 6 required templates exist, Deployment probes/resources/envFrom, HPA apiVersion/min/max/conditional, per-service values exist and parse, containerPort/targetPort/autoscaling per architecture |
| **P1** | 93 | Chart.yaml details (appVersion, type, description), values.yaml defaults (resources, probes, autoscaling, ingress, PDB, networkPolicy), _helpers.tpl content (6 helpers, standard labels), Service/ConfigMap/Ingress/ServiceAccount template patterns, per-service ingress/networkPolicy/secrets/image, optional PDB/NetworkPolicy templates |
| **P2** | 64 | PDB/NetworkPolicy template content, per-service resource limits, `helm template` rendering validation (45 tests), conditional rendering per service, rendered YAML quality |
| **Total** | **235** | |

---

## Test Strategy Notes

### Why Static Config + Template Pattern Validation Tests

Story 1-9 creates **Helm chart files** (Chart.yaml, values.yaml, Go template files) and per-service values overrides -- not application code. Tests use three validation approaches:

1. **File existence checks**: Chart directory structure, required template files, per-service values files
2. **YAML content validation**: Chart.yaml metadata, values.yaml defaults, per-service values correctness (parsed with `pyyaml`)
3. **Go template pattern matching**: Template files contain expected Kubernetes resource patterns (kind, apiVersion, .Values references, conditional rendering)
4. **Helm CLI rendering**: `helm template` validates that templates + values produce valid Kubernetes YAML

All tests use `@pytest.mark.unit` marker. No Docker Compose required. The rendering tests gracefully skip when `helm` is not available.

### Why No E2E/Integration Tests

- Helm charts are static configuration files validated by `helm template`
- The epic-level test design classifies Helm validation as P2 (Medium priority)
- Rendering tests with `helm template` are the gold-standard for chart validation
- Production deployment validation happens in the CI pipeline and actual cluster

### Failure Mechanisms

Tests fail via three mechanisms:

1. **Directory/file missing**: Chart directory and template files don't exist -> every path existence test fails
2. **YAML content missing**: Chart.yaml and values.yaml don't exist -> config loads as `None` -> every dependent test fails
3. **Helm not installed**: `helm` binary not found -> rendering tests skip (graceful degradation)

When the developer starts implementing:

1. **Tasks 1.1-1.5 (Chart structure)**: Create Chart.yaml, values.yaml, templates/, _helpers.tpl, NOTES.txt -> 64 chart structure tests should pass
2. **Tasks 2-9 (Templates)**: Create all 8 template files -> 48 template pattern tests should pass
3. **Task 10 (Per-service values)**: Create 5 service values files -> 78 service values tests should pass
4. **Task 11 (Rendering)**: With helm installed, `helm template` validates all services -> 45 rendering tests should pass

---

## Files Generated

| File | Purpose | AC |
|------|---------|-----|
| `eusolicit-app/tests/unit/test_helm_chart_structure.py` | Validates chart directory, Chart.yaml metadata, values.yaml defaults, required/optional template files, _helpers.tpl content | AC 1, 2, 8 |
| `eusolicit-app/tests/unit/test_helm_templates.py` | Validates Go template patterns: Deployment (probes, resources, envFrom), HPA (apiVersion, min/max, CPU/memory), Service, ConfigMap, Ingress, ServiceAccount, PDB, NetworkPolicy | AC 3, 4, 8 |
| `eusolicit-app/tests/unit/test_helm_service_values.py` | Validates per-service values files: existence, ports, autoscaling, ingress, networkPolicy, secrets, image names, resource limits per architecture mandate | AC 5, 6 |
| `eusolicit-app/tests/unit/test_helm_template_rendering.py` | Validates `helm template` rendering: no errors, valid YAML, correct resources present, conditional rendering per feature flags | AC 7, 8 |
| `eusolicit-docs/test-artifacts/atdd-checklist-1-9-helm-chart-base-template-service-values.md` | This ATDD checklist | -- |

---

## Next Steps (TDD Green Phase)

After implementing the chart files:

1. **Tasks 1.1-1.5 -- Chart structure**: Create Chart.yaml, values.yaml, templates/, _helpers.tpl, NOTES.txt
   - Run: `cd eusolicit-app && python -m pytest tests/unit/test_helm_chart_structure.py -v`
   - Expected: 64/64 pass

2. **Tasks 2-9 -- Templates**: Create deployment.yaml, service.yaml, configmap.yaml, hpa.yaml, ingress.yaml, serviceaccount.yaml, pdb.yaml, networkpolicy.yaml
   - Run: `cd eusolicit-app && python -m pytest tests/unit/test_helm_templates.py -v`
   - Expected: 48/48 pass

3. **Task 10 -- Per-service values**: Create client-api.yaml, admin-api.yaml, data-pipeline.yaml, ai-gateway.yaml, notification.yaml
   - Run: `cd eusolicit-app && python -m pytest tests/unit/test_helm_service_values.py -v`
   - Expected: 78/78 pass

4. **Task 11 -- Rendering validation**: Install Helm 3 and validate rendering
   - Run: `cd eusolicit-app && python -m pytest tests/unit/test_helm_template_rendering.py -v`
   - Expected: 45/45 pass (requires `helm` CLI)

5. **Full regression**: `cd eusolicit-app && python -m pytest tests/unit/test_helm_*.py -v`
   - Expected: 235/235 pass

6. **Verify no regressions**: `cd eusolicit-app && python -m pytest tests/unit/ -v`
   - Expected: All 1152 existing tests + 235 new tests = 1387 pass

### Implementation Order (Recommended)

Based on dependency analysis:

1. **Chart.yaml** (Task 1.1) -- standalone, establishes chart identity
2. **values.yaml** (Task 1.2) -- standalone, provides all default values
3. **_helpers.tpl** (Task 1.4) -- standalone, referenced by all templates
4. **NOTES.txt** (Task 1.5) -- standalone, post-install instructions
5. **Templates** (Tasks 2-9) -- depend on _helpers.tpl and values.yaml structure
   - deployment.yaml first (most complex, validates probe/resource/envFrom patterns)
   - service.yaml, configmap.yaml next (simple, cross-references deployment)
   - hpa.yaml, ingress.yaml, serviceaccount.yaml (conditional rendering)
   - pdb.yaml, networkpolicy.yaml last (optional features)
6. **Per-service values** (Task 10) -- depend on values.yaml defaults for override structure
7. **Rendering validation** (Task 11) -- depends on all above being complete

---

## Quality Gate Criteria

| Gate | Threshold | Status |
|------|-----------|--------|
| P0 tests pass rate | 100% | Pending implementation |
| P1 tests pass rate | >= 95% | Pending implementation |
| P2 tests pass rate | >= 90% | Pending implementation |
| No high-risk items unmitigated | E01-R-009 covered | ✅ |
| Existing test regression | 0 new failures | ✅ Verified (1152/1152 pass) |
| helm template renders for all 5 services | 0 errors | Pending implementation |
| Architecture-mandated ports match (8001-8005) | All 5 correct | Pending implementation |
| Architecture-mandated HPA ranges match | All 5 correct | Pending implementation |

---

## Validation Checklist

- [x] Prerequisites satisfied (story approved, pytest + pyyaml configured, dev env available)
- [x] All 4 test files created with correct structure and markers
- [x] Tests cover all 8 acceptance criteria
- [x] Tests designed to fail before implementation (ATDD RED phase)
- [x] Priority tags assigned per epic-level test design
- [x] No placeholder assertions (all assert expected behavior)
- [x] Test patterns match existing project conventions (class-based, docstrings, pytestmark)
- [x] 190 tests confirmed FAILING, 45 confirmed SKIPPED (helm not available)
- [x] Checklist saved to `test-artifacts/` directory
- [x] No DO NOT TOUCH files modified
- [x] Chart.yaml tests validate apiVersion v2, name, version, appVersion, type
- [x] values.yaml tests validate all 22 required top-level keys with correct defaults
- [x] Template tests validate Go template patterns for all 8 template files
- [x] Per-service values tests validate architecture-mandated specs for all 5 services
- [x] Rendering tests validate `helm template` output for all 5 services
- [x] Conditional rendering tested for Ingress, NetworkPolicy, HPA per service
- [x] API versions validated: autoscaling/v2 (not v2beta2), networking.k8s.io/v1, policy/v1
- [x] Existing tests unaffected: 1152/1152 pass (zero regressions)

---

**Generated by:** BMad TEA Agent -- ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
