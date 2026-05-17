# E26: Agent-Driven Ingestion & Analysis

**Sprint:** post-pivot S+2..S+3 | **Points:** 34 | **Dependencies:** E04 amendment, E05 amendment, E24, E25, E28 | **Milestone:** SirmaAI Pivot

> **Source:** `sprint-change-proposal-2026-05-12-sirmaai.md`, `prd-amendment-2026-05-12-sirmaai.md` (FR-15 rewrite, FR-47 qualification, FR-48 quantification), `architecture-amendment-2026-05-12-sirmaai.md` (ADR-018, ADR-019).

## Goal

Realise the user's stated principle — *"all data ingestion flows are dominated by the AI agents in SirmaAI, for qualification, quantification analysis, as well as further processing."* Author N8N workflow templates that own the end-to-end opportunity lifecycle: ingestion (AOP/TED/EU Grants) → qualification (fit assessment, gap analysis, pursue/monitor/decline recommendation) → quantification (effort, win probability, expected value) → optional CRM enrichment via MCP tools (E27) → user-visible scored opportunity with structured agent outputs.

E05 amendment owns the ingestion plumbing (workflow templates calling crawler agents + webhook receiver writing canonical opportunities). E26 owns **the broader narrative**: making sure qualification and quantification are first-class, exposed via API, triggered on right events (new opportunity, status transition, user request), and structured for UI consumption. Long-running analyses use SirmaAI async-run (`run-async` + `jobs/{jobId}/status` polling per E04 amendment S04.24). Idempotency via run-ID dedup.

## Acceptance Criteria

- [ ] Three SirmaAI agents seeded in Project template (per E24 S24.03) for the analysis pipeline: `opportunity_qualifier`, `opportunity_quantifier`, `opportunity_post_processor` (configurable per tenant via SirmaAI agent versions)
- [ ] N8N workflow template `opportunity-analysis-v1` orchestrates: receive opportunity ID → fetch canonical row from `pipeline.opportunities` → invoke qualifier (async-run) → on `pursue` outcome invoke quantifier (async-run) → invoke post-processor → emit structured result via `agent.run.completed` webhook
- [ ] Qualification agent returns structured payload: `fit_score (0-100)`, `gap_analysis (list of strings)`, `recommended_action ∈ {pursue, monitor, decline}`, `confidence (0-1)`, `kb_citations (list)`
- [ ] Quantification agent returns structured payload: `estimated_effort_person_days`, `estimated_win_probability (0-1)`, `expected_value_eur`, `recommended_bid_threshold`, `kb_citations (list)`
- [ ] EU Solicit endpoint `POST /api/v1/opportunities/{id}/analyze` triggers the workflow on user demand; returns `run_id`; UI polls status or subscribes to SSE for completion
- [ ] Automatic trigger: on new opportunity ingestion (`opportunities.ingested` event in E05), qualification fires automatically for each new opportunity; quantification fires only on `recommended_action='pursue'`
- [ ] Results persisted to `client.opportunity_analyses` (new table — see schema below) and surfaced in opportunity detail UI
- [ ] All agent runs traceable: `gateway.workflow_runs` row written via E04 S04.24; SirmaAI traces accessible via deep-link from EU Solicit admin UI
- [ ] Idempotency: re-running analysis with same opportunity ID + analysis type within 1h returns existing result unless `force=true`
- [ ] Long-running analyses (qualification typically <30s, quantification can be 1-3 min): async-run + job poll; UI shows progress with `aria-live` updates
- [ ] Tier-gated: qualification available all tiers; quantification gated Pro+ (existing TierGate Depends)
- [ ] Cross-tenant negative: analysis triggered for tenant A's opportunity cannot be invoked under tenant B's Project context
- [ ] KB grounding: when KB has relevant artefacts (company profile, past proposals), agent outputs include citations; UI surfaces them per E25 S25.09

## New Schema

```sql
-- =====================================================================
-- OPPORTUNITY ANALYSIS RESULTS (E26)
-- =====================================================================

CREATE TABLE client.opportunity_analyses (
    id                      UUID PRIMARY KEY,
    company_id              UUID NOT NULL REFERENCES client.companies(id),
    opportunity_id          UUID NOT NULL,            -- references pipeline.opportunities; no FK (schema isolation)
    analysis_type           TEXT NOT NULL,            -- qualification | quantification | post_processing
    sirmaai_run_id          TEXT NOT NULL,
    eusolicit_run_id        UUID NOT NULL UNIQUE REFERENCES gateway.workflow_runs(eusolicit_run_id),
    status                  TEXT NOT NULL,            -- pending | completed | failed
    result_payload          JSONB,                    -- structured agent output
    kb_citations            JSONB DEFAULT '[]',       -- array of {file_id, passage_excerpt, relevance}
    started_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at            TIMESTAMPTZ,
    error_message           TEXT
);
CREATE INDEX ix_opportunity_analyses_company_oppty
    ON client.opportunity_analyses (company_id, opportunity_id, analysis_type)
    WHERE status = 'completed';
```

## Stories

### S26.01: `opportunity_qualifier` SirmaAI agent definition + Project template addition
**Points:** 3 | **Type:** backend + prompt-engineering
**Prerequisite (added 2026-05-15):** `eusolicit-docs/test-artifacts/e26-baseline-datasets-spec.md` § "Qualifier Baseline" curated and committed at `services/client-api/tests/data/sirmaai-baselines/v1/qualifier-baseline-v1.json`. The eval-run AC below is unmeasurable without this dataset.

Author the qualification agent in SirmaAI Project template (added to S24.03 seed): system prompt covering fit assessment against company profile, gap analysis methodology, pursue/monitor/decline decision criteria. Tool bindings: KB search (for retrieving company profile + past proposals + qualification rubrics). Structured output schema (JSON). Eval-runs harness in SirmaAI with 20 sample opportunities for prompt regression testing.

**Acceptance:**
- Agent definition committed to `services/client-api/config/sirmaai_project_template.yaml`
- Eval-run with 20 sample opportunities (per `qualifier-baseline-v1.json`) → ≥85% align with human qualification baseline (alignment metric defined in baseline spec: 0.6·action-match + 0.3·fit-score-band + 0.1·gap-keyword-overlap)
- Structured output schema validated by Pydantic on EU Solicit side

---

### S26.02: `opportunity_quantifier` SirmaAI agent definition + template
**Points:** 3 | **Type:** backend + prompt-engineering
**Prerequisite (added 2026-05-15):** `eusolicit-docs/test-artifacts/e26-baseline-datasets-spec.md` § "Quantifier Baseline" curated and committed at `services/client-api/tests/data/sirmaai-baselines/v1/quantifier-baseline-v1.json` (15 opps with historical actuals + 2 synthetic thin-KB adversarial opps).

Quantification agent: effort estimation, win probability, expected value. Tool bindings: KB search, MCP CRM tools (E27, for past-deal-history if available). Structured output schema. Eval-runs harness with 15 sample opportunities + historical outcomes.

**Acceptance:**
- Agent committed to template
- Eval-run: estimated effort within ±25% AND estimated expected-value within ±25% of historical actuals on validation set (accuracy metric defined in baseline spec)
- For the 2 synthetic thin-KB opps: agent MUST return `confidence ≤ medium` AND `caveat: "low_kb_context"` (overconfidence is a fail signal)
- Quantification respects "if no historical CRM data, fall back to KB-grounded estimate with lower confidence" pattern

---

### S26.03: `opportunity-analysis-v1` N8N workflow template
**Points:** 5 | **Type:** workflow + backend

Author N8N workflow template orchestrating the analysis pipeline (see Goal). Parameterised by `(projectId, opportunityId, force, analysis_type)`. Calls SirmaAI agents async-run; polls for completion; emits result via webhook. Semver-tagged; PR-reviewed; rollback plan documented. Staged-rollout discipline (per ADR-018): canary tenant → 10% → 100% on new version.

**Acceptance:**
- Template runs end-to-end in staging with sample opportunity
- Versioning: `opportunity-analysis-v1` lives alongside any future `-v2`; per-tenant flag selects version
- Rollback runbook tested: flip tenant feature flag back to `-v1` → next analysis uses old version

---

### S26.04: `client.opportunity_analyses` schema + service layer
**Points:** 3 | **Type:** backend

Alembic migration + SQLAlchemy model. Service functions: `start_analysis(company_id, opportunity_id, analysis_type, force=False)`, `get_latest_completed(opportunity_id, analysis_type)`, `record_completion(eusolicit_run_id, payload, kb_citations)`. Idempotency: re-running within 1h returns existing row unless force=True.

**Acceptance:**
- Migration up + down clean
- Idempotency test: 10 concurrent `start_analysis` calls → 1 actual run, 10 return same `eusolicit_run_id`
- Index hit verified by EXPLAIN ANALYZE

---

### S26.05: User-trigger endpoint `POST /api/v1/opportunities/{id}/analyze`
**Points:** 3 | **Type:** backend

Endpoint accepts `{ "analysis_type": "qualification" | "quantification" | "both", "force": false }`. Tier-gated (quantification Pro+). Calls `sirmaai-gateway.trigger_workflow("opportunity-analysis-v1", projectId, opportunityId, ...)`. Returns `run_id` + 202 Accepted. UI polls `GET /api/v1/opportunities/{id}/analyses` for results or subscribes to SSE.

**Acceptance:**
- 202 with `run_id` returned within 200ms p95
- Workspace-scope check + cross-tenant negative test
- Tier-gate 402 for Professional tenant requesting quantification

---

### S26.06: Auto-trigger on `opportunities.ingested`
**Points:** 3 | **Type:** backend

Subscriber on Redis Stream `opportunities.ingested`: for each new opportunity, dispatch `start_analysis(company_id, opportunity_id, "qualification")`. On qualification completion (webhook), if `recommended_action='pursue'`, auto-trigger quantification (tier-permitting). Throttle: per tenant max 100 concurrent in-flight analyses to avoid SirmaAI rate-limit (gauge against `sirmaai_concurrent_runs`).

**Acceptance:**
- New opportunity → qualification within 60s (steady state)
- pursue → quantification cascade triggers correctly
- Throttle prevents cascading 429s on bulk ingestion

---

### S26.07: Webhook-driven result persistence + SSE notification
**Points:** 3 | **Type:** backend + frontend

Webhook handler subscribes to `sirmaai.agent.run.completed` from `sirmaai-gateway` (per E28 routing). On match for `client.opportunity_analyses` row: parse result payload, validate against expected structured schema, persist, set `completed_at`. Frontend SSE channel `/api/v1/opportunities/{id}/analyses/stream` pushes completion events to opportunity-detail UI.

**Acceptance:**
- Webhook → DB row update → SSE push within 5s p95
- Payload schema mismatch → `failed` status + structured error in `error_message`
- SSE lifecycle invariants (ADR-005) respected — quota check before stream, fresh session, terminal event

---

### S26.08: Opportunity-detail UI for analysis results
**Points:** 5 | **Type:** frontend + integration

Update opportunity-detail page: qualification panel (fit score gauge, gap analysis list, recommendation badge), quantification panel (effort, win prob, expected value, threshold), citations strip (links to KB files per E25 S25.09). Loading states with progress, `aria-live` for completion. Re-run button (with tier check on quantification). Empty state for "no analysis yet — trigger one".

**Acceptance:**
- WCAG 2.1 AA: keyboard nav, `aria-live` for status changes, sufficient color contrast for fit score gauge
- Playwright E2E: opportunity ingested → auto-qualification → UI shows result → user clicks "Run quantification" → result appears
- TanStack Query invalidation on webhook completion (via SSE subscription)

---

### S26.09: KB-grounding regression tests + structured-output validation
**Points:** 3 | **Type:** integration

Integration test suite (mock SirmaAI responses) that asserts:
- Qualification grounds in company profile + past proposals when present
- Quantification grounds in historical CRM data when available (mocked MCP response)
- Citations populated correctly with `client.sirmaai_kb_files.id` references
- Schema validation fails fast on malformed agent output (no silent partial results)

**Acceptance:**
- 20+ scenarios covering happy path, malformed output, missing KB, mid-stream cancel
- Snapshot tests for citation format (frozen contract for frontend rendering)

---

### S26.10: Equivalence harness vs. legacy E11 quantification (if any)
**Points:** 3 | **Type:** integration + ops

Pre-pivot, E11's relevance scoring + grant eligibility offered partial qualification. New qualification covers a broader scope but should not regress on overlap. Author a 50-opportunity validation set; run old + new in parallel for one sprint; flag deltas where new agent decisions diverge from old without justification. Used to tune prompts before promoting `opportunity-analysis-v1` to 100% rollout.

**Acceptance:**
- 50 opportunities run through both paths
- Delta report distinguishes "improvement" from "regression"
- Sign-off: human reviewer accepts new path before 100% rollout

## Salvaged patterns

- Async-run + job-poll pattern (E04 amendment S04.24).
- Structured agent output validation via Pydantic (Epic 4 pattern).
- SSE lifecycle (ADR-005, hardened Epic 13).
- TanStack Query invalidation via SSE (Epic 6 pattern).
- TierGate Depends (ADR-006).

## Out of scope

- Comparative analysis across multiple opportunities (e.g. "rank my 50 saved opportunities by expected value") — separate epic post-launch.
- Manual override of agent recommendations (e.g. user marks an opportunity "actually pursue" when agent said decline) — pattern exists in E07 proposal-review flow; reuse there.
- Continuous agent re-evaluation on KB updates (re-running historical analyses when profile changes) — costly; defer to user-triggered re-run.
