---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-18'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 7
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-06.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-05.md'
  - 'eusolicit-app/services/client-api (router scaffolding)'
  - '_bmad/bmm/config.yaml'
---

# Test Design Progress — Epic 7

## Step 1: Mode Detection

- **Mode:** Epic-Level (explicit user intent; epic file supplied)
- **Epic:** E07 — Proposal Generation & Document Intelligence
- **Epic file:** `eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md`
- **Prerequisite check:** Epic + 16 stories with ACs; system-level test design available (`test-design-architecture.md`, `test-design-qa.md`); config loaded from `_bmad/bmm/config.yaml`.

## Step 2: Context Loaded

- Config: `test_artifacts = eusolicit-docs/test-artifacts`, `planning_artifacts = eusolicit-docs/planning-artifacts`
- Stack: fullstack (FastAPI Python 3.12 + Next.js 14)
- System-level risks inherited: R-01 (multi-tenancy), R-02 (billing), R-03 (AI hallucinations), R-06 (prompt injection); new E07 surfaces: rich-text export, multi-agent orchestration, version history, rollback atomicity
- Knowledge fragments referenced: `risk-governance.md`, `probability-impact.md`, `test-levels-framework.md`, `test-priorities-matrix.md`
- Prior epic test-design reviewed: E06 (tier gating, SSE, ClamAV); E05 (pipeline schema)

## Step 3: Risk & Testability

- 12 risks identified (5 high-priority ≥6, 5 medium, 2 low)
- Key testability concerns: non-determinism of KraftData agents; rich-text HTML → export injection surface; SSE mid-stream persistence; optimistic-locking semantics for section auto-save

## Step 4: Coverage Plan

- P0: 12 tests (~25–40h)
- P1: 32 tests (~35–50h)
- P2: 16 tests (~14–22h)
- P3: 6 tests (~4–7h)
- Total: 66 tests, ~78–119h (~2–3 weeks, 1 QA)

## Step 5: Output generated

- Final document: `eusolicit-docs/test-artifacts/test-design-epic-07.md`
