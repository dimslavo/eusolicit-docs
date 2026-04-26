---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-19'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 9
outputDocuments:
  - 'eusolicit-docs/test-artifacts/test-design-epic-09.md'
inputDocuments:
  - 'eusolicit-docs/EU_Solicit_PRD_v1.md'
  - 'eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md'
  - 'eusolicit-docs/EU_Solicit_Service_Decomposition_Analysis.md'
  - 'eusolicit-docs/EU_Solicit_Requirements_Brief_v4.md'
  - 'eusolicit-docs/project-context.md'
  - 'eusolicit-docs/architecture-vs-requirements-gap-analysis.md'
outputDocuments:
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/test-design/eu-solicit-handoff.md'
---

# System-Level Test Design Progress — EU Solicit

**Run Date:** 2026-04-19
**Mode:** System-Level (v2 — full replacement of 2026-04-12 documents)

---

## Step 1: Detect Mode

**Mode selected:** System-Level (explicitly specified in arguments; PRD + Architecture docs confirmed available)

**Reason for v2 run:** The 2026-04-12 documents contained critical architecture inaccuracies:
- Referenced **Kafka** (not present — platform uses **Redis 7 Streams**)
- Referenced **Elasticsearch** (not present — platform uses **PostgreSQL 16 FTS**)
- Referenced **"AI Solicit Service" / direct OpenAI/Anthropic** (not present — platform uses **KraftData AI Gateway** with 29 agents)
- Only had 6 risks and 16 test scenarios; all new features (ESPD, tasks, approvals, collaboration, per-bid add-ons, EU VAT) were absent

---

## Step 2: Load Context

**Stack detected:** fullstack (Python 3.12 FastAPI backend + Next.js 14 frontend)

**Key artifacts loaded:**
- PRD v1 — 10 feature areas, 5 personas, NFRs (99.5% availability, <200ms API latency, GDPR/EU data residency)
- Architecture v4 — 5-service SOA, 6 PostgreSQL schemas, Redis 7 Streams (7 streams/4 consumer groups), KraftData 29 agents, Stripe/SendGrid/MinIO/ClamAV
- Requirements Brief v4 — Pricing model (Free/Starter/Professional/Enterprise), Stripe billing details, KraftData integration patterns
- project-context.md — 52 implementation rules, 126 deferred items, anti-patterns (== HMAC, sync ORM, Celery workers)
- Gap analysis — 15 gaps resolved in v4 (ESPD, tasks, approvals, collaboration, per-bid add-ons, VAT, content blocks)

**TEA config flags:** No explicit TEA config found in BMad config; defaults applied:
- `tea_execution_mode`: sequential
- `tea_use_playwright_utils`: not set (Python stack primary — pytest/eusolicit-test-utils)
- `tea_use_pactjs_utils`: not set

---

## Step 3: Risk Assessment

**Total risks identified:** 18

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | P | I | Score |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **R-001** | TECH | KraftData AI Gateway: non-deterministic outputs + circuit breaker complexity | 2 | 3 | **6** |
| **R-002** | SEC | Cross-tenant isolation failure (two-tier RBAC edge cases) | 2 | 3 | **6** |
| **R-003** | PERF | SSE connection pool saturation under peak AI usage | 2 | 3 | **6** |
| **R-004** | BUS | Billing inaccuracy: trial/add-on/VAT/usage-metering overlap in Stripe | 2 | 3 | **6** |
| **R-006** | SEC | Entity-level RBAC bypass on per-proposal collaborator permissions | 2 | 3 | **6** |
| **R-018** | TECH/BUS | ESPD XML serialisation non-conformance to EU XSD v2.1+ | 2 | 3 | **6** |

### Medium-Priority Risks (Score 3–5)

R-005 (DATA, 4), R-007 (TECH, 4), R-008 (PERF, 4), R-010 (SEC, 3), R-011 (TECH, 4), R-012 (BUS, 4), R-013 (OPS, 3), R-014 (SEC, 3), R-015 (SEC, 4), R-017 (SEC, 4)

### Low-Priority Risks (Score ≤2)

R-009 (OPS, 2), R-016 (TECH, 2)

---

## Step 4: Coverage Plan

**Total test scenarios:** 32

| Priority | Count | Description |
| :--- | :--- | :--- |
| P0 | 8 | Critical paths: registration, JWT lifecycle, cross-tenant isolation, tier gating, SSE AI streaming, Stripe trial lifecycle, ClamAV upload security, HMAC webhook |
| P1 | 12 | Two-tier RBAC matrix, entity permissions, usage metering, per-bid add-ons, EU VAT, circuit breaker, section locking, AI draft SSE, Redis Streams delivery, ESPD auto-fill, audit trail, approval workflow |
| P2 | 8 | DLQ, digest assembly, PDF/DOCX export, materialized views, iCal, task orchestration, content blocks, submission guides |
| P3 | 4 | Visual regression, i18n completeness, FTS performance, white-label |

**Execution strategy:** Every PR (P0+P1 in ~15 min sharded) / Nightly (P2 + perf baselines) / Weekly (security, chaos, DAST)

**Quality gates:** P0 = 100% pass | P1 ≥ 95% | Coverage ≥ 80% | P95 API latency < 200ms

---

## Step 5: Generate Output

**Mode resolved:** sequential (no agent-team/subagent capability detected; config default)

**Documents generated:**

| Document | Path | Status |
| :--- | :--- | :--- |
| Architecture Test Design (v2) | `test-artifacts/test-design-architecture.md` | ✅ Written 2026-04-19 |
| QA Test Design (v2) | `test-artifacts/test-design-qa.md` | ✅ Written 2026-04-19 |
| BMAD Handoff (v1.4) | `test-artifacts/test-design/eu-solicit-handoff.md` | ✅ Updated 2026-04-19 |

**Key corrections vs. 2026-04-12:**
- ✅ Redis 7 Streams (not Kafka)
- ✅ PostgreSQL FTS (not Elasticsearch)
- ✅ KraftData AI Gateway (not "AI Solicit Service")
- ✅ 18-risk register (not 6 risks)
- ✅ 32 test scenarios (not 16)
- ✅ eusolicit-test-utils Python examples (not TypeScript Playwright)
- ✅ All new features covered: ESPD, tasks, approvals, collaboration, per-bid add-ons, EU VAT, content blocks, submission guides

**Open assumptions:**
1. KraftData mock service (TB-02) must be built before AI test suite can execute
2. Stripe CLI in CI (TB-03) required before billing tests can run in automated pipeline
3. Test data seeding API (TB-01) required before parallel E2E execution
4. Redis Streams DLQ (TB-04) must be deployed for DLQ test scenarios

---

## E09 Epic-Level Test Design Run — 2026-04-19

**Mode:** Epic-Level | **Epic:** E09 — Notifications, Alerts & Calendar | **Sprint:** 9–10 | **Points:** 55

### Step 1: Mode Detection
Epic-level mode confirmed from explicit user argument. Epic file loaded: `epic-09-notifications-alerts-calendar.md`. Prerequisites confirmed: epic + stories with full acceptance criteria available; architecture context available.

### Step 2: Context Loaded
Stack: fullstack (Python Celery + Next.js 14). System-level test designs loaded for context. 14 stories spanning Celery worker scaffold, notification schema, alert preferences CRUD, Redis Stream consumers, SendGrid email, iCal feed, Google + Microsoft OAuth2 calendar sync, task/approval consumers, trial expiry, Stripe usage, 2 frontend pages, PDF report generation.

### Step 3: Risk Assessment
12 risks identified. 4 high-priority (score ≥6):
- E09-R-001 (SEC, 6): Fernet calendar token key exposure
- E09-R-002 (DATA, 6): Redis consumer group duplicate dispatch on redelivery
- E09-R-003 (SEC, 6): SendGrid webhook signature bypass
- E09-R-004 (DATA, 6): Stripe usage counter atomicity (crash between report + DEL)
6 medium (score 3–4): Digest window edge case, cache staleness, calendar sync diff, Graph throttling, Beat timezone, trial event coupling
2 low (score 1–2): Digest N+1, iCal token in logs

### Step 4: Coverage Plan
150 total tests across 4 priorities:
- P0: 33 tests (~16–28h) — commit gate
- P1: 71 tests (~28–48h) — PR gate
- P2: 41 tests (~12–20h) — nightly
- P3: 5 tests (~3–6h) — on-demand
Total effort: ~59–102 hours (~1.5–2.5 weeks, 1 QA)

### Step 5: Output Generated
Document written: `eusolicit-docs/test-artifacts/test-design-epic-09.md`

Open assumptions for E09:
1. E06 `client.alert_preferences` + `client.calendar_connections` schemas frozen before Sprint 9 starts
2. SendGrid dynamic template IDs available before S09.06 ATDD
3. Google/Microsoft OAuth test credentials provisioned before Sprint 10 calendar sync testing
4. E05 `opportunities.ingested` event payload schema stable (shared `eusolicit-models` envelope)
