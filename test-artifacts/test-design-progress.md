---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-12'
workflowType: 'testarch-test-design'
mode: 'epic-level'
lastEpic: 12
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md'
  - 'knowledge/risk-governance.md'
  - 'knowledge/probability-impact.md'
  - 'knowledge/test-levels-framework.md'
  - 'knowledge/test-priorities-matrix.md'
---

# Test Design Progress — EU Solicit (System-Level)

## Step 1: Mode Detection

**Mode selected:** System-Level
**Reason:** User explicitly requested system-level test design. PRD + ADR + Architecture documents available.
**Prerequisites met:** PRD v1 (FRs + NFRs), ADRs 1.1–1.15, Solution Architecture v4, User Journeys v1.

## Step 2: Context Loading

**Configuration:**
- `test_stack_type`: auto → detected as `fullstack` (playwright.config.ts + pyproject.toml + package.json)
- `tea_use_playwright_utils`: not configured (disabled)
- `tea_use_pactjs_utils`: not configured (disabled)
- `tea_pact_mcp`: not configured (disabled)
- `tea_browser_automation`: not configured (disabled)
- `test_artifacts`: `eusolicit-docs/test-artifacts`

**Project artifacts loaded:**
- PRD v1 — 10 feature areas, 4 NFRs, 3 release milestones
- Requirements Brief v4 — pricing model, tier access controls, agent inventory (29 agents), vector stores (7)
- Solution Architecture v4 — 15 ADRs, 5 services, 6 DB schemas, inter-service communication matrix, 16 event types
- User Journeys v1 — 5 personas, 4+ journey maps with UX requirements

**Knowledge fragments loaded (system-level required):**
- `adr-quality-readiness-checklist.md` — 8 categories, 29 criteria
- `test-levels-framework.md` — E2E vs API vs Unit decision matrix
- `risk-governance.md` — P×I scoring, gate decisions, mitigation tracking
- `test-quality.md` — Definition of Done (no hard waits, <300 lines, <1.5 min)

**Tech stack extracted:**
- Backend: Python 3.12, FastAPI, Celery, SQLAlchemy 2.0, Pydantic 2.x, httpx
- Frontend: Next.js 14, React 18, Tailwind, shadcn/ui, Tiptap, Zustand, TanStack Query
- Database: PostgreSQL 16 (6 schemas), Redis 7
- Testing: pytest + pytest-asyncio (backend), Playwright (E2E/API)
- Infrastructure: Docker, Kubernetes, Helm, GitHub Actions, Terraform

## Step 3: Testability & Risk Assessment

**Testability review completed:**
- Controllability: 2 blockers (no seeding API, no KraftData mock), 2 improvements needed
- Observability: Strong (OpenTelemetry + Jaeger + Prometheus + Grafana + Loki); gap in correlation ID propagation
- Reliability: Strong (circuit breaker, schema isolation, stateless services); concern about Redis Streams dead letter

**Risk assessment completed:**
- Total risks: 15
- High-priority (≥6): 4 (R-001, R-002, R-004, R-006) — plus R-003 at borderline
- Medium (3–5): 7
- Low (1–2): 4
- All high-priority risks have mitigation strategies, owners, and timelines assigned

## Step 4: Coverage Plan

**Coverage matrix completed:**
- P0: ~18 tests (auth, billing, data isolation, core AI flows)
- P1: ~25 tests (opportunity lifecycle, proposals, collaboration, alerts)
- P2: ~30 tests (admin features, edge cases, analytics, calendar)
- P3: ~15 tests (E2E journeys, performance benchmarks, i18n)
- Total: ~88 test scenarios

**Execution strategy:** PR (~10–15 min) / Nightly (k6 perf) / Weekly (chaos/DR)
**Quality gates:** P0 = 100%, P1 ≥ 95%, high-risk mitigations before release, coverage ≥ 80%
**Effort estimate:** ~6–9 weeks (1 QA), ~3–5 weeks (2 QAs)

## Step 5: Output Generation

**Documents generated:**
1. `eusolicit-docs/test-artifacts/test-design-architecture.md` — Architecture concerns, risks, testability gaps
2. `eusolicit-docs/test-artifacts/test-design-qa.md` — Test execution recipe with full coverage plan
3. `eusolicit-docs/test-artifacts/test-design/eu-solicit-handoff.md` — BMAD integration handoff

**Execution mode:** Sequential (no subagent/agent-team capability detected)
**Validation:** Checked against checklist.md — all required sections populated.

## Completion Report

- **Mode:** System-Level
- **Output files:** 3 documents (architecture, QA, handoff)
- **Key risks:** R-001 (KraftData SPOF, score 9), R-002 (multi-tenant isolation, 6), R-004 (Stripe billing, 6), R-006 (entity-level RBAC, 6)
- **Gate thresholds:** P0 = 100%, P1 ≥ 95%, coverage ≥ 80%
- **Open assumptions:** KraftData 99.5% SLA, Stripe Tax handles all EU VAT, all 29 agents pre-configured before E2E
- **Pre-implementation blockers:** 3 (seeding API, KraftData mock, Stripe test config)

---

## Epic-Level Run: Epic 12 (2026-04-12)

### Step 1: Mode Detection

**Mode selected:** Epic-Level
**Reason:** User explicitly requested epic-level test design for Epic 12.
**Prerequisites met:** Epic 12 file with 18 stories and acceptance criteria ✅; system-level test design (test-design-architecture.md, test-design-qa.md) ✅; ATDD P0 checklist (atdd-checklist-e12-p0.md, 19 tests, RED phase) ✅

### Step 2: Context Loading

**Configuration:**
- `test_stack_type`: auto → detected `fullstack`
- All tea_* settings: not configured (disabled)
- `test_artifacts`: `eusolicit-docs/test-artifacts`

**Project artifacts loaded:**
- Epic 12: 18 stories (S12.01–S12.18), 55 story points, Sprint 13–14
- System-level test design: 18 risks (6 high-priority, score ≥6), 3 testability blockers (TB-01–TB-03)
- Prior ATDD P0 checklist: 19 atomic tests in 4 spec files, RED phase complete (2026-04-05)

**Knowledge fragments loaded (epic-level required):**
- `risk-governance.md` — P×I scoring, gate decisions
- `probability-impact.md` — 1–3 scale definitions
- `test-levels-framework.md` — E2E vs API vs Unit decision matrix
- `test-priorities-matrix.md` — P0–P3 criteria

**Coverage gaps from existing tests:** P0 ATDD complete; P1–P3 not yet generated

### Step 3: Risk Assessment

**Risks identified for Epic 12:**
- Total: 11
- High-priority (score ≥6): 3 (E12-R-001 SEC/6, E12-R-002 BUS/6, E12-R-003 SEC/6)
- Medium (score 3–5): 6 (E12-R-004–R-009)
- Low (score 1–2): 2 (E12-R-010–R-011)

**Key risks:**
- E12-R-001: Cross-tenant analytics leakage via materialized views (P=2, I=3, Score=6)
- E12-R-002: Tier gate bypass on Professional+ dashboards (P=2, I=3, Score=6)
- E12-R-003: OWASP audit false confidence before launch (P=2, I=3, Score=6)

### Step 4: Coverage Plan

**Coverage matrix:**
- P0: 9 scenarios → 19 atomic tests (17 Playwright + 2 infra); ~15–25 hours
- P1: 18 tests; ~20–35 hours
- P2: 20 tests; ~8–15 hours
- P3: 4 tests; ~2–5 hours
- Total: ~61 functional + 2 infrastructure; ~45–80 hours (~1.5–2.5 weeks, 1 QA)

**Execution strategy:** PR (functional ~10–15 min) / Nightly (k6) / Weekly (ZAP)
**Quality gates:** P0=100%, P1≥95%, all SEC risks mitigated, load test p95 < 500ms

### Step 5: Output Generation

**Document generated:**
- `eusolicit-docs/test-artifacts/test-design-epic-12.md` (v3, 2026-04-12)
  - Status: Final — P0 ATDD Red Phase Complete; P1–P3 ATDD Pending
  - 11 risks documented (3 high, 6 medium, 2 low)
  - 51 scenarios → ~61 atomic tests
  - Full mitigation plans for E12-R-001, R-002, R-003
  - Interworking and regression scope for 6 services

**Execution mode:** Sequential
**Validation:** Checked against checklist.md — all required sections populated and compliant

**Completion:**
- Mode: Epic-Level (Epic 12)
- Output file: `eusolicit-docs/test-artifacts/test-design-epic-12.md` (v3)
- Key risks: E12-R-001 (cross-tenant analytics, 6), E12-R-002 (tier gate bypass, 6), E12-R-003 (OWASP confidence, 6)
- Gate thresholds: P0=100%, P1≥95%, SEC coverage 100%
- ATDD P0 RED phase: complete (19 tests in 4 files)
- Next step: GREEN phase as S12.01–S12.15 endpoints land in staging; run `*atdd` for P1–P3
