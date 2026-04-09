---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-09'
workflowType: 'testarch-test-design'
mode: 'epic-level'
lastEpic: 11
inputDocuments:
  - 'eusolicit-docs/EU_Solicit_PRD_v1.md'
  - 'eusolicit-docs/EU_Solicit_Requirements_Brief_v4.md'
  - 'eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md'
  - 'eusolicit-docs/EU_Solicit_User_Journeys_and_Workflows_v1.md'
  - 'eusolicit-docs/planning-artifacts/epic-03-frontend-shell-design-system.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
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

## System-Level: Regenerated 2026-04-07

**Reason:** Full system-level test design re-run. Architecture context refreshed against Solution Architecture v4.

**Bugs corrected from prior version:**
- R-003 (SSE stream saturation, score 6) was classified as Medium in the QA doc — corrected to **High**
- R-006 (entity-level RBAC, score 6) was classified as Medium in the architecture doc — corrected to **High**
- Agent/team/workflow count corrected from "29" to **22** (per Requirements Brief §7 and Architecture §11.2)
- Risk summary updated from "4 high-priority risks" to **5 high-priority risks** (R-001=9, R-002=6, R-003=6, R-004=6, R-006=6)
- Handoff document updated: R-003 added to epic-level risk references (E04 quality gate)

**Changes from prior version:**
- Architecture v4 context: 7 Redis Streams + 4 consumer groups explicitly documented in risk register (R-007)
- Dependencies table updated with per-milestone sprint numbers
- Companion doc reference added at bottom of architecture doc

**Output files updated:**
1. `eusolicit-docs/test-artifacts/test-design-architecture.md` — v2 (2026-04-07)
2. `eusolicit-docs/test-artifacts/test-design-qa.md` — v4.1 (2026-04-07)
3. `eusolicit-docs/test-artifacts/test-design/eu-solicit-handoff.md` — v1.1 (2026-04-07)

**Execution mode:** Sequential
**Validation:** Checked against checklist.md — all required sections populated; risk classifications corrected.

---

## Epic-Level: E12 — Analytics, Reporting & Admin Platform

### Step 1: Mode Detection

**Mode selected:** Epic-Level
**Reason:** User explicitly requested epic-level test design for Epic 12. Epic and story requirements with acceptance criteria available.
**Prerequisites met:** Epic file with 18 stories and acceptance criteria; system-level test designs (architecture + QA) available for context.

### Step 2: Context Loading

**Configuration:** Inherited from system-level run (fullstack, Playwright + pytest, test_artifacts at eusolicit-docs/test-artifacts).

**Artifacts loaded:**
- Epic 12: 18 stories (S12.01–S12.18), 55 points, Sprints 13–14
- System-level test design (architecture doc) — risk register, testability concerns
- System-level test design (QA doc) — coverage plan, execution strategy
- Knowledge fragments: risk-governance, probability-impact, test-levels-framework, test-priorities-matrix

**Existing test coverage scanned:**
- E2E: 3 spec files (auth/login, opportunities/search, smoke/health)
- Backend: 4 Python test files (Orchestrator: playbooks, config, phase order, rate limit)
- No existing analytics/admin/enterprise API tests

### Step 3: Risk & Testability Assessment

**Epic-level risks identified:** 11 total
- High-priority (score >= 6): 3 (E12-R-001: cross-tenant analytics leakage, E12-R-002: tier gate bypass, E12-R-003: OWASP audit false confidence)
- Medium (3–5): 6
- Low (1–2): 2
- System-level risk inheritance: E12-R-001 extends R-002, E12-R-002 extends R-012

### Step 4: Coverage Plan

**Coverage matrix:** 52 test scenarios
- P0: 10 tests (cross-tenant isolation, tier gates, admin/enterprise auth, security, load testing)
- P1: 18 tests (materialized views, analytics APIs, report generation, admin CRUD, enterprise API)
- P2: 20 tests (frontend E2E, secondary admin features, API edge cases)
- P3: 4 tests (onboarding wizard, empty states, error pages)

**Execution strategy:** PR (all functional ~10–15 min) / Nightly (k6 perf) / Weekly (ZAP security)
**Quality gates:** P0 = 100%, P1 >= 95%, SEC tests = 100%, p95 < 500ms
**Effort estimate:** ~45–80 hours (~1.5–2.5 weeks, 1 QA)

### Step 5: Output Generation

**Document generated:**
- `eusolicit-docs/test-artifacts/test-design-epic-12.md` — Epic-level test design for E12

**Execution mode:** Sequential
**Validation:** Checked against checklist.md — all required sections populated.

### Completion Report

- **Mode:** Epic-Level
- **Epic:** E12 — Analytics, Reporting & Admin Platform
- **Output file:** `test-design-epic-12.md`
- **Key risks:** E12-R-001 (cross-tenant analytics, 6), E12-R-002 (tier gate bypass, 6), E12-R-003 (OWASP coverage, 6)
- **Gate thresholds:** P0 = 100%, P1 >= 95%, SEC = 100%, p95 < 500ms
- **Open assumptions:** Materialized views populated before testing; E05–E08 stable; SendGrid sandbox supports attachments
- **Dependencies:** TB-01 seeding API, TB-02 AI Gateway mock, SendGrid sandbox, k6 runner, OWASP ZAP

---

## Epic-Level: E12 — Regenerated 2026-04-06

**Reason:** Re-run of epic-level test design workflow for E12.
**Changes from prior version:** Refreshed date; validated all sections against checklist.md; confirmed all 52 test scenarios, 11 risks, and 3 high-priority mitigations remain current.
**Output file:** `eusolicit-docs/test-artifacts/test-design-epic-12.md`

---

## Epic-Level: E01 — Infrastructure & Monorepo Foundation

### Step 1: Mode Detection

**Mode selected:** Epic-Level
**Reason:** User explicitly requested epic-level test design for Epic 1. Epic file with 10 stories and acceptance criteria available.
**Prerequisites met:** Epic file with 10 stories (S01.01–S01.10), 34 points, Sprints 1–2; system-level test designs (architecture + QA) available for context.

### Step 2: Context Loading

**Configuration:** Inherited from system-level run (fullstack, Playwright + pytest, test_artifacts at eusolicit-docs/test-artifacts).

**Artifacts loaded:**
- Epic 1: 10 stories (S01.01–S01.10), 34 points, Sprints 1–2
- System-level test design (architecture doc) — risk register, testability concerns
- System-level test design (QA doc) — coverage plan, execution strategy
- Knowledge fragments: risk-governance, probability-impact, test-levels-framework, test-priorities-matrix

**Existing test coverage scanned:**
- E2E: 7 Playwright specs (auth/login, opportunities/search, smoke/health, analytics/cross-tenant, analytics/tier-gate, admin/access-control, enterprise/api-auth)
- Backend: 5 Python test files (Orchestrator: playbooks, config, phase order, rate limit, story seeding)
- Per-service test infrastructure: conftest.py + empty unit/integration/api directories for all 5 services
- Shared test utilities: eusolicit-test-utils package (factories, ServiceClient, DB helpers, Redis helpers, JWT generation)
- No existing infrastructure/monorepo-specific tests

### Step 3: Risk & Testability Assessment

**Epic-level risks identified:** 11 total
- High-priority (score >= 6): 2 (E01-R-001: PostgreSQL schema isolation enforcement, E01-R-002: Redis Streams dead letter handling)
- Medium (3–5): 6 (Docker Compose instability, Alembic migration ordering, shared package install breakage, CI matrix timeout, event serialization inconsistency, middleware contract misalignment)
- Low (1–2): 3 (Helm rendering, ClamAV startup delay, Terraform drift)
- System-level risk inheritance: E01-R-001 extends R-002 (multi-tenant isolation), E01-R-002 extends R-007 (Redis Streams delivery)

### Step 4: Coverage Plan

**Coverage matrix:** 31 test scenarios
- P0: 5 tests (schema isolation, Docker Compose healthy, Redis Streams smoke, shared package imports, CI green)
- P1: 11 tests (negative access, migrations, DLQ, streams/groups, eusolicit-common modules, eusolicit-models DTOs/events, eusolicit-kraftdata models)
- P2: 11 tests (Docker profiles, MinIO, ClamAV, Makefile, Helm, logging, middleware, Redis idempotent, CI cache)
- P3: 4 tests (Terraform validate, Next.js placeholder, Docker image size, CI timing)

**Execution strategy:** PR (all functional ~10–15 min) / On merge (full CI matrix)
**Quality gates:** P0 = 100%, P1 >= 95%, SEC (schema isolation) = 100%, shared package coverage >= 80%
**Effort estimate:** ~28–51 hours (~1–1.5 weeks, 1 QA)

### Step 5: Output Generation

**Document generated:**
- `eusolicit-docs/test-artifacts/test-design-epic-01.md` — Epic-level test design for E01

**Execution mode:** Sequential
**Validation:** Checked against checklist.md — all required sections populated.

### Completion Report

- **Mode:** Epic-Level
- **Epic:** E01 — Infrastructure & Monorepo Foundation
- **Output file:** `test-design-epic-01.md`
- **Key risks:** E01-R-001 (schema isolation enforcement, 6), E01-R-002 (Redis Streams DLQ, 6)
- **Gate thresholds:** P0 = 100%, P1 >= 95%, SEC = 100%, shared package coverage >= 80%
- **Open assumptions:** Docker Compose v2 available; PostgreSQL/Redis Docker images support required features; GitHub Actions has sufficient runner resources
- **Dependencies:** Docker images (PostgreSQL 16, Redis 7, MinIO, ClamAV), GitHub Actions, Python 3.12

---

## Epic-Level: E02 — Authentication & Identity

### Step 1: Mode Detection

**Mode selected:** Epic-Level
**Reason:** User explicitly requested epic-level test design for Epic 2. Epic file with 12 stories and acceptance criteria available.
**Prerequisites met:** Epic file with 12 stories (S02.01–S02.12), 34 points, Sprints 1–2; system-level test designs (architecture + QA) available for context.

### Step 2: Context Loading

**Configuration:** Inherited from system-level run (backend stack, pytest + pytest-asyncio, test_artifacts at eusolicit-docs/test-artifacts).

**Artifacts loaded:**
- Epic 2: 12 stories (S02.01–S02.12), 34 points, Sprints 1–2
- System-level test design (architecture doc) — R-002 (multi-tenant isolation), R-006 (entity RBAC), TB-01 blocker
- System-level test design (QA doc) — P0-001..P0-007, P0-018, P1-022 coverage entries
- Knowledge fragments: risk-governance, probability-impact, test-levels-framework, test-priorities-matrix

**Existing test coverage scanned:**
- Unit tests: 28 files in eusolicit-app/tests/unit/ (E01 infra only)
- Per-service test scaffolds: 5 service conftest.py + empty unit/integration/api directories
- No existing auth or RBAC tests found

### Step 3: Risk & Testability Assessment

**Epic-level risks identified:** 9 total
- High-priority (score >= 6): 3 (E02-R-001: RBAC ceiling enforcement, E02-R-002: cross-tenant isolation, E02-R-003: token security)
- Medium (3–5): 4 (OAuth CSRF, password security, Redis rate limit leakage, audit log completeness)
- Low (1–2): 2 (ESPD schema gaps, invite token lifecycle)
- System-level risk inheritance: E02-R-001 extends R-006, E02-R-002 extends R-002

### Step 4: Coverage Plan

**Coverage matrix:** 52 test scenarios
- P0: 12 tests (registration, JWT validation, RBAC matrix, cross-tenant isolation, token rotation/replay)
- P1: 18 tests (duplicate email, rate limiting, OAuth flows, password reset, team management, audit trail)
- P2: 18 tests (migration validation, CRUD edge cases, ESPD versioning, audit log enforcement)
- P3: 4 tests (E2E auth chain, OAuth E2E, password reset chain, k6 auth latency)

**Execution strategy:** PR (all functional ~10–15 min) / Nightly (k6 perf) / Weekly (security regression)
**Quality gates:** P0 = 100%, P1 >= 95%, SEC tests = 100%, branch coverage >= 90%
**Effort estimate:** ~50–86 hours (~1.5–2.5 weeks, 1 QA)

### Step 5: Output Generation

**Document generated:**
- `eusolicit-docs/test-artifacts/test-design-epic-02.md` — Epic-level test design for E02

**Execution mode:** Sequential
**Validation:** Checked against checklist.md — all required sections populated.

### Completion Report

- **Mode:** Epic-Level
- **Epic:** E02 — Authentication & Identity
- **Output file:** `test-design-epic-02.md`
- **Key risks:** E02-R-001 (RBAC ceiling, 6), E02-R-002 (cross-tenant isolation, 6), E02-R-003 (token security, 6)
- **Gate thresholds:** P0 = 100%, P1 >= 95%, SEC = 100%, branch coverage >= 90%
- **Open assumptions:** E01 complete; RSA key pair generated before Sprint 1; authlib OAuth mock strategy agreed; email sending stubbed
- **Dependencies:** TB-01 seeding API, RSA key pair fixture, authlib mock, Redis Docker Compose, Python 3.12

---

## Epic-Level: E03 — Frontend Shell & Design System

### Step 1: Mode Detection

**Mode selected:** Epic-Level
**Reason:** User explicitly requested epic-level test design for Epic 3. Epic file with 12 stories and acceptance criteria available.
**Prerequisites met:** Epic file with 12 stories (S03.01–S03.12), 37 points, Sprints 1–2; system-level test designs (architecture + QA) available for context.

### Step 2: Context Loading

**Configuration:** Inherited from system-level run (fullstack, Playwright + pytest, test_artifacts at eusolicit-docs/test-artifacts).

**Artifacts loaded:**
- Epic 3: 12 stories (S03.01–S03.12), 37 points, Sprints 1–2
- System-level test design (architecture doc) — R-013 (i18n), risk register, testability concerns
- System-level test design (QA doc) — coverage plan, execution strategy, tool selection (Playwright + pytest + k6)
- E02 test design — E02-R-003 (token security) inherited as E03-R-003 at frontend layer
- Knowledge fragments: risk-governance, probability-impact, test-levels-framework, test-priorities-matrix

**Tech stack extracted:**
- Frontend: Next.js 14 App Router, React 18, Tailwind CSS, shadcn/ui, Zustand, TanStack Query v5, React Hook Form + Zod, next-intl v3
- Monorepo: pnpm workspaces + Turborepo; packages/ui + packages/config shared
- Testing: Playwright (E2E), Vitest/Jest (unit), k6 (performance)
- Detected stack: frontend

**Existing test coverage scanned:**
- E2E: 7 Playwright specs (auth/login, opportunities/search, smoke/health, analytics/*3, admin/access-control, enterprise/api-auth)
- Backend: 5+ Python test files
- No existing frontend shell or design system tests

### Step 3: Risk & Testability Assessment

**Epic-level risks identified:** 10 total
- High-priority (score >= 6): 3 (E03-R-001: locale middleware routing, E03-R-002: route guard flash, E03-R-003: 401 refresh deduplication)
- Medium (3–5): 4 (Zustand hydration mismatch, responsive breakpoint regression, missing i18n keys, wizard state loss)
- Low (1–2): 3 (CSS variable conflicts, SSE EventSource leak, dark mode flash)
- System-level risk inheritance: E03-R-001 extends R-013 (i18n, score 2 → 6 at epic), E03-R-003 extends E02-R-003 (token security)

### Step 4: Coverage Plan

**Coverage matrix:** 52 test scenarios
- P0: 10 tests (build gate, route guard redirects, hydration flash, locale routing, 401 dedup, app shell render)
- P1: 18 tests (Zustand persistence, auth form validation, i18n key coverage, wizard full flow, responsive layout, apiClient, FormField, toast, skeleton, error boundary)
- P2: 20 tests (component renders, breakpoint edge cases, SSE cleanup, wizard step edge cases, empty states, dark mode, middleware redirect)
- P3: 4 tests (dark mode, sidebar animation, k6 performance)

**Execution strategy:** PR (all functional ~10–15 min) / Nightly (k6 perf) / Weekly (device QA, animation visual)
**Quality gates:** P0 = 100%, P1 >= 95%, SEC (route guard) = 100%, build hard-fail gate
**Effort estimate:** ~50–86 hours (~1.5–2.5 weeks, 1 QA)

### Step 5: Output Generation

**Document generated:**
- `eusolicit-docs/test-artifacts/test-design-epic-03.md` — Epic-level test design for E03

**Execution mode:** Sequential
**Validation:** Checked against checklist.md — all required sections populated.

### Completion Report

- **Mode:** Epic-Level
- **Epic:** E03 — Frontend Shell & Design System
- **Output file:** `test-design-epic-03.md`
- **Key risks:** E03-R-001 (locale routing, 6), E03-R-002 (route guard flash, 6), E03-R-003 (401 dedup, 6)
- **Gate thresholds:** P0 = 100%, P1 >= 95%, SEC (route guard) = 100%, build hard-fail, branch coverage >= 80%
- **Open assumptions:** E01 complete; auth endpoints stubbed via MSW or Playwright route(); BG placeholder translations cover all key namespaces; Playwright ≥1.40 compatible with Next.js 14 App Router
- **Dependencies:** E01 monorepo scaffold, MSW or Playwright route() auth stubs, packages/ui barrel exports, CPV JSON for wizard, both apps start in CI

---

## Epic-Level: E03 — Re-validated 2026-04-09

**Reason:** Epic-level test design workflow re-run in autopilot mode. Existing `test-design-epic-03.md` validated against current epic file (12 stories, S03.01–S03.12), system-level architecture doc (v3, 2026-04-09), and checklist.md.

**Findings:**
- All 12 stories covered by 52 test scenarios (P0=10, P1=18, P2=20, P3=4) ✅
- S03.12 (Route Guards) fully covered: E03-P0-002/003/004, E03-P1-017/018, E03-P2-018/019 ✅
- 10 risks validated as correct and evidence-based ✅
- 3 high-priority risks (E03-R-001/002/003, all score 6) have complete mitigation plans ✅
- System-level R-016/R-017 (added 2026-04-09) map to E03-R-001/E03-R-003 respectively — no conflict, E03 scores correctly higher (6 vs 4) ✅
- All checklist criteria met: risk scoring, coverage priorities, execution strategy (PR/Nightly/Weekly), resource estimates as ranges, quality gates, entry/exit criteria, not-in-scope, interworking ✅
- No changes required to output document

**Output file:** `eusolicit-docs/test-artifacts/test-design-epic-03.md` — unchanged, current, complete
**Execution mode:** Sequential
**Validation:** Full checklist pass — no gaps identified

---

## Epic-Level: E03 — Re-generated 2026-04-09 (autopilot)

**Reason:** Full epic-level workflow re-run in BMAD autopilot mode. All 5 steps executed fresh against current epic file and system-level docs (architecture v3, 2026-04-09).

**Steps executed:**
- Step 1: Mode Detection → Epic-Level confirmed (user explicit; epic file with 12 stories loaded)
- Step 2: Context Loaded → Epic 3 (S03.01–S03.12, 37 pts), system arch doc (v3, R-016/R-017 present), QA doc (v5), E02 test design (token security inheritance); stack = fullstack; no playwright-utils/pact config
- Step 3: Risk & Testability → 10 risks re-validated: 3 high (E03-R-001/002/003, all score=6), 4 medium, 3 low; system R-016/R-017 correctly scored lower (4) than E03 epic-level equivalents (6)
- Step 4: Coverage Plan → 52 scenarios confirmed: P0=10, P1=18, P2=20, P3=4; S03.12 (Route Guards) fully covered across P0+P1+P2 tiers
- Step 5: Output → `test-design-epic-03.md` date updated to 2026-04-09; all content verified correct; no substantive changes required

**Checklist validation results:**
- Risk scoring: All 10 risks have correct P×I scores ✅
- Coverage priorities: 52 scenarios across 12 stories ✅
- Execution strategy: Simple PR/Nightly/Weekly with philosophy note ✅
- Resource estimates: All as interval ranges ✅
- Quality gates: P0=100%, P1≥95%, SEC=100%, build hard-fail, branch ≥80% ✅
- Entry/Exit criteria: Both defined ✅
- Not in scope: 8 items with reasoning and mitigation ✅
- Interworking: 3 upstream/downstream dependencies documented ✅
- Assumptions: 5 documented; Dependencies: 5 with required dates ✅
- Follow-on: atdd + automate workflows referenced ✅
- Priority note at top of coverage plan: present ✅

**Output file:** `eusolicit-docs/test-artifacts/test-design-epic-03.md` — updated date to 2026-04-09, content verified complete
**Execution mode:** Sequential
**Validation:** Full checklist pass — all criteria met

---

## System-Level: Regenerated 2026-04-09

**Reason:** Full system-level test design re-run. Architecture v4 context (2026-04-05) fully loaded. Incorporates learnings from epic-level designs E01, E02, E03, and E12.

**Changes from prior version (2026-04-07):**
- **Agent count corrected to 29** (27 agents + 1 team + 1 workflow per Architecture §11.2 table) — previous run had an error correcting to 22; architecture v4 table is authoritative at 29 entries
- **Risk register expanded to 17 risks** (previously 15): added R-016 (locale middleware routing, score 4) and R-017 (JWT refresh race at frontend, score 4) — surfaced from E03 epic-level analysis
- **Medium-priority risk count:** 8 (previously 6)
- **Coverage updated to ~92 tests** (previously ~88): P0=20, P1=27, P2=30, P3=15 — 4 new tests for R-016 and R-017 (P1-024 through P1-027)
- **Pre-implementation blocker TB-04 added:** Redis Streams DLQ (per consumer group + monitoring) — surfaced from E01 epic-level analysis as blocking event-driven flow tests
- **data-testid table added** to handoff document for frontend story testability requirements
- **Per-epic quality gate table** updated in handoff document: E03 now includes locale routing + JWT refresh gates

**Output files updated:**
1. `eusolicit-docs/test-artifacts/test-design-architecture.md` — v3 (2026-04-09)
2. `eusolicit-docs/test-artifacts/test-design-qa.md` — v5 (2026-04-09)
3. `eusolicit-docs/test-artifacts/test-design/eu-solicit-handoff.md` — v1.2 (2026-04-09)

**Key risks:** R-001 (KraftData SPOF, score 9), R-002 (multi-tenant isolation, 6), R-003 (SSE saturation, 6), R-004 (Stripe billing, 6), R-006 (entity RBAC, 6)
**New medium risks:** R-016 (locale routing, 4), R-017 (JWT refresh race, 4)
**Gate thresholds:** P0 = 100%, P1 ≥ 95%, coverage ≥ 80%, REST p95 < 200ms, SSE TTFB < 500ms
**Open assumptions:** KraftData 29 entities pre-configured; Stripe Tax handles all EU VAT; OAuth test accounts available
**Pre-implementation blockers:** 4 (TB-01 seeding API, TB-02 KraftData mock, TB-03 Stripe test config, TB-04 Redis DLQ)

**Execution mode:** Sequential
**Validation:** Checked against checklist.md — all required sections populated; agent count corrected; new risks classified and mitigated.

---

## Epic-Level: E11 — EU Grant Specialization & Compliance

### Step 1: Mode Detection

**Mode selected:** Epic-Level
**Reason:** User explicitly requested epic-level test design for Epic 11. Epic file with 16 stories and acceptance criteria available.
**Prerequisites met:** Epic file (S11.01–S11.16, 55 pts, Sprints 11–12); system-level test designs (architecture v3, QA v5) available for context.

### Step 2: Context Loading

**Configuration:** Inherited from system-level run (fullstack, Playwright + pytest + playwright-utils, test_artifacts at eusolicit-docs/test-artifacts).

**Artifacts loaded:**
- Epic 11: 16 stories (S11.01–S11.16), 55 points, Sprints 11–12, MVP milestone
- System-level architecture doc (v3, 2026-04-09) — risk register (17 risks), TB-01 through TB-04, R-001 (KraftData SPOF, score 9)
- System-level QA doc (v5, 2026-04-09) — coverage plan, execution strategy
- Knowledge fragments: risk-governance, probability-impact, test-levels-framework, test-priorities-matrix

**Key tech context extracted:**
- 8 new KraftData agent types introduced: Grant Eligibility, Budget Builder, Consortium Finder, Logframe Generator, Reporting Template Generator, ESPD Auto-Fill, Framework Suggestion, Regulation Tracker
- EU ESPD XML schema compliance (Parts II-V) is a novel test requirement not present in prior epics
- Celery Beat scheduling for Regulation Tracker Agent requires dedicated test harness

**Existing test coverage scanned:**
- No prior E11-specific tests found
- ESPD Profile CRUD (S02.12 from E02) covered in atdd-checklist-2-12-espd-profile-crud.md — S11 extends this schema

### Step 3: Risk & Testability Assessment

**Epic-level risks identified:** 12 total
- High-priority (score ≥ 6): 5 (E11-R-001: AI Gateway error handling, E11-R-002: ESPD XML conformance, E11-R-003: admin endpoint auth, E11-R-004: budget arithmetic, E11-R-005: ESPD RLS)
- Medium (3–5): 5 (E11-R-006 through E11-R-010)
- Low (1–2): 2 (E11-R-011: DOCX perf, E11-R-012: Consortium result quality)
- System-level risk inheritance: E11-R-001 extends R-001 (KraftData SPOF), E11-R-003 extends R-006 (RBAC), E11-R-005 extends R-002 (multi-tenant isolation)

### Step 4: Coverage Plan

**Coverage matrix:** 56 test scenarios
- P0: 10 tests (AI Gateway mock validation, ESPD RLS, ESPD XML XSD, budget arithmetic, admin 403, agent timeout/retry)
- P1: 18 tests (full agent CRUD, ESPD CRUD, compliance framework CRUD, suggestion flows, Regulation Tracker, deletion guard)
- P2: 20 tests (API edge cases + Playwright frontend panels)
- P3: 8 tests (5 × S11.16 E2E journeys, DOCX validation, k6 agent load)

**Execution strategy:** PR (P0+P1+P2 ~10–15 min) / Nightly (P3 E2E journeys + k6) / Weekly (ESPD XSD conformance regression, DOCX visual)
**Quality gates:** P0 = 100%, P1 ≥ 95%, SEC = 100%, ESPD XML XSD validation gate, agent error handling gate
**Effort estimate:** ~65–115 hours (~2–3.5 weeks, 1 QA)

### Step 5: Output Generation

**Document generated:**
- `eusolicit-docs/test-artifacts/test-design-epic-11.md` — Epic-level test design for E11

**Execution mode:** Sequential
**Validation:** Checked against checklist.md — all required sections populated.

### Completion Report

- **Mode:** Epic-Level
- **Epic:** E11 — EU Grant Specialization & Compliance
- **Output file:** `test-design-epic-11.md`
- **Key risks:** E11-R-001 (AI Gateway error handling, 6), E11-R-002 (ESPD XML conformance, 6), E11-R-003 (admin auth, 6), E11-R-004 (budget arithmetic, 6), E11-R-005 (ESPD RLS, 6)
- **Gate thresholds:** P0 = 100%, P1 ≥ 95%, SEC = 100%, ESPD XSD validation gate, agent error handling gate
- **Open assumptions:** EU ESPD XSD v2.1+ publicly accessible; all 8 KraftData agent types pre-configured before staging E2E; TB-02 extended for all 8 new agent types; Celery Beat test trigger available
- **Dependencies:** TB-01 (ESPD/framework seeding), TB-02 (8 new agent mocks + failure injection), EU ESPD XSD, Celery Beat test trigger, E04/E06/E07 stable
