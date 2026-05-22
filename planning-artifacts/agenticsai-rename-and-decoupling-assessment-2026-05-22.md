# Assessment: AgenticSAI rename + de-coupling

**Date:** 2026-05-22
**Status:** Draft for review — no code/doc renames executed yet
**Author:** Engineering (assessment), pending architecture sign-off
**Supersedes terminology of:** `architecture-amendment-2026-05-12-sirmaai.md`, `prd-amendment-2026-05-12-sirmaai.md`

---

## 1. Purpose & decisions

Initiative: eliminate every reference to the platform's legacy names — **KraftData**
(original commercial name) and **SirmaAI** (interim internal codename) — in favour of the
real, current brand **AgenticSAI** (`https://agenticsai.endigitalx.com/`), across both the
codebase and the documentation; and re-shape the integration so AgenticSAI is a **point of
choice, not a strict dependency**, with **strict architectural isolation** treating
AgenticSAI as a third-party SaaS.

Decisions taken at kick-off (2026-05-22):

| # | Decision | Choice | Consequence |
|---|----------|--------|-------------|
| D1 | What "point of choice" means | **Both optional + swappable** | EU Solicit must run with AI fully disabled (no-AI mode) AND a provider abstraction must allow replacing AgenticSAI with another AI SaaS. |
| D2 | How deep the rename reaches | **Full rename incl. DB + env** | Also rename DB schema/role/table/column names and env vars; requires Alembic migrations, prod `.env` changes on www1, and a maintenance window. Zero legacy references anywhere. |
| D3 | Immediate deliverable | **Persist assessment + phased plan** | This document. No code changes until the plan is approved. |

**Brand reality (confirmed from disk):** AgenticSAI is not a new name to invent — the
authoritative architecture/PRD amendments and the platform's own OpenAPI contract already use
"AgenticSAI", and `eusolicit-docs/AgenticsAI-reference-docs/` is already correctly named. This
initiative *finishes* a rename the platform itself completed; "SirmaAI" was always an internal
stand-in.

---

## 2. Naming surface inventory

Scale is large but the **risk is concentrated**, not uniform. Search tokens (case-insensitive):
`sirmaai`, `sirma.ai`, `kraftdata`, `kraft_data`, `ai-gateway`, `ai_gateway`, `aigw`,
`agenticsai`.

| Category | Volume | Rename mechanism | Risk |
|---|---|---|---|
| Docs: ADRs, PRD, epics, runbooks, impl-artifacts (~300 files) | ~6k refs | Text replace; ADRs/PRD already say "AgenticSAI" | Low |
| Python imports / modules / classes / comments / tests | thousands | Mechanical per-package rename | Low |
| Package distributions: `eusolicit-sirmaai` (← `eusolicit-kraftdata`) | 68 imports | Single-commit rename, all consumers in one go | Low–Med |
| Service names: `ai-gateway` → `sirmaai-gateway` → `agenticsai-gateway` (+ `-worker`, `-beat`) | dozens | compose + Dockerfile + Makefile + Helm in lockstep | Medium |
| **Env vars / config keys**: `SIRMAAI_*`, `AIGW_*`, `CLIENT_API_AIGW_BASE_URL`, `ADMIN_API_AIGW_BASE_URL`, `AI_GATEWAY_BASE_URL`, `KRAFTDATA_*` | hundreds | settings classes + prod `.env` on www1 | **High** |
| **DB schema / role / table / column**: `gateway` schema, `ai_gateway_role`, `client.sirmaai_projects`, `sirmaai_project_id`, Celery task names (`sirmaai_*`) | ~150 refs | Alembic migration + infra init script + maintenance window | **High** |
| External hostnames | few | None — already `agenticsai.endigitalx.com` | None |
| **Dead trees**: `services/ai-gateway/`, `packages/eusolicit-kraftdata/` | — | Delete | Low (but dev compose still runs `ai-gateway` — see §3) |

**Authority tiers for docs** (rename order): Tier 1 authoritative — `PRD.md`,
`prd-amendment-2026-05-12-sirmaai.md`, `architecture.md`, `architecture-amendment-2026-05-12-sirmaai.md`
(tracked edits, review required). Tier 2 — epic specs (E04, E24–E28). Tier 3 — implementation
artifacts, test artifacts, retrospectives (bulk replace, no deep review). Filenames containing
`sirmaai`/`kraftdata`/`ai-gateway` get renamed too (e.g. `kraftdata-outage.md` →
`agenticsai-outage.md`).

---

## 3. Architecture & coupling findings

### 3.1 Integration anatomy

```
            EU SOLICIT CORE                       |  3rd-PARTY SaaS
                                                  |
 client-api ─┐  HTTP (thin client, internal)      |
 admin-api  ─┼─────────────▶  agenticsai-gateway ──┼──▶ AgenticSAI
 data-pipe  ─┘                (FastAPI facade /     |    agenticsai.endigitalx.com
   │                           anti-corruption layer)|   (per-project bearer, webhooks)
   │ raw SQL read                  │ reads/writes    |
   ▼                               ▼                 |
 client.sirmaai_projects      gateway schema         |
 (tenant identity + keys       (workflow_runs,        |
  live in CORE schema —        webhook_dlq, subs,     |
  LEAK)                        rate_limit_audit)      |

 eusolicit-sirmaai (DTO-only) — imported ONLY by the gateway, never by domain code.
```

The gateway **is** an anti-corruption layer. Consumers (`client_api/services/ai_gateway_client.py`,
`admin_api/core/ai_gateway.py`) only speak HTTP to it and receive plain dicts. No EU Solicit
service makes a direct AgenticSAI call or imports its DTOs.

### 3.2 Isolation scorecard

| Dimension | Verdict | Note |
|---|---|---|
| Domain code calls AgenticSAI directly | **Well isolated** | Never; HTTP to internal gateway only |
| AgenticSAI DTOs in domain models | **Well isolated** | `eusolicit-sirmaai` imported only by gateway; consumers use dicts |
| Gateway facade quality | **Good** | Exception translation, circuit breaker, rate limiter, webhook receiver + DLQ |
| Credential / network boundary | **Good** | Per-project Fernet keys, 90-day rotation, webhook HMAC via `compare_digest` |
| Schema isolation | **Leaky** | Tenant identity/credentials in `client` schema; 1 cross-schema FK; 1 raw cross-schema read |
| Provider neutrality | **Leaky** | No `AIProvider` abstraction; `kraftdata_id` / `call_kraftdata` / `n8n_subdomain` vocabulary throughout the gateway |
| Can run without it (optionality) | **Leaky** | Gateway always boots & loads registry; no "AI-off" mode; flag-OFF still hits legacy KraftData host |
| Rename hygiene | **Leaky** | Dev compose runs retired `ai-gateway`; consumer default base URL `http://ai-gateway:8000` |

### 3.3 Strict-dependency points (file:line, severity)

| # | Point | Location | Severity |
|---|---|---|---|
| 1 | Startup fail-fast: `init_registry()` runs unconditionally, raises if `config/agents.yaml` missing | `sirmaai_gateway/main.py:81` → `agent_registry.py:196` | **High** — this is the 2026-05-22 prod crash-loop |
| 2 | Startup fail-fast: `RuntimeError` if flag-on and no `SIRMAAI_FERNET_KEY` | `main.py:73-76` | Medium |
| 3 | Startup fail-fast: `get_tier_rate_limits()` unconditional | `main.py:101-104` | Medium |
| 5 | AgenticSAI tenant state in `client` schema (`client.sirmaai_projects`) | `client_api/models/sirmaai_project.py:28-34` | **High** |
| 6 | Cross-schema FK `gateway.workflow_runs.company_id → client.companies` | `models/workflow_run.py:55-62` | Medium |
| 7 | data-pipeline raw-SQL read of `client.sirmaai_projects` | `data_pipeline/services/workflow_event_consumer.py:147-151` | Medium |
| 8 | Consumer default base URL names retired service (`http://ai-gateway:8000`) | `client-api/config.py:111` | **High (drift)** |
| 9 | Dev compose builds retired `services/ai-gateway/Dockerfile` | `docker-compose.yml:254-265` | **High (drift)** |
| 10 | Makefile `up`/migrate loops reference `ai-gateway` (dev) vs `sirmaai-gateway` (prod) | `Makefile:25,121` vs `:182` | Medium (drift) |
| 11 | No provider abstraction — gateway is AgenticSAI-shaped end to end | `execution.py`, `agent_registry.py` | High (blocks swappability) |

**Key nuance:** flag-OFF ≠ "no AI". Today it falls back to the legacy KraftData host
(`kraftdata_base_url` default `https://stage.sirma.ai`). There is no fully-AI-disabled mode.
Non-AI features keep working when AgenticSAI is down (per-request `503 AGENT_UNAVAILABLE`,
fail-closed degraded banner), but every AI feature hard-depends on the gateway being up.

---

## 4. Target end-state

**Naming convention:**
- `AgenticSAI` — brand, docs, user-facing strings.
- `agenticsai` — code identifiers, module/package names, service names, env-var prefix, DB schema/role/columns.
- Neutral terms (`ai_provider`, `provider_id`, `run_agent`) — at the abstraction layer, so the
  boundary is provider-agnostic and a second provider needs no renaming.

**Architecture (per D1 both optional + swappable, D2 full rename):**
1. **`AIProvider` interface** (Protocol/ABC): `run_agent`, `submit_async`, `poll_status`,
   `resolve_agent`, `register_webhook`. `AgenticSAIProvider` is one implementation; a `NoopProvider`
   backs the no-AI mode. Provider selected by `AI_PROVIDER` config (`agenticsai` | `none`).
2. **No-AI mode** (`AI_PROVIDER=none`): consumers short-circuit to `503 AGENT_UNAVAILABLE` / hide
   AI features without a gateway hop; gateway startup becomes non-fatal (registry/rate-limit/Fernet
   loads gated behind provider selection).
3. **Schema isolation**: relocate tenant state from `client.sirmaai_projects` →
   `agenticsai` (renamed `gateway`) schema as `agenticsai.projects`; replace data-pipeline's raw
   read and client-api's direct ORM access with a gateway API call; hold an opaque `company_id`
   (drop or formalize the cross-schema FK, mirroring `rate_limit_sync_audit`).
4. **Full identifier rename**: `gateway` schema → `agenticsai`; `ai_gateway_role` →
   `agenticsai_role`; `sirmaai_project_id` → `agenticsai_project_id`; `SIRMAAI_*`/`AIGW_*` env →
   `AGENTICSAI_*`; Celery tasks `sirmaai_*` → `agenticsai_*`.

---

## 5. Phased plan

Lowest-risk first. Each phase is an independent PR (or PR set). The autonomous orchestrator runs
on the same `eusolicit-app` working tree (currently mid-Epic 11) — **phases touching code must be
coordinated with / pause the orchestrator** (it commits to `main`), and DB/prod phases need a
maintenance window on www1.

### P0 — Drift cleanup *(unblocks deploys; do first)*
- Fix dev `docker-compose.yml` to build `sirmaai-gateway` (not retired `ai-gateway`); align
  `Makefile` service lists; fix consumer default base URLs (`client-api/config.py:111`, admin equiv).
- Delete dead trees: `services/ai-gateway/`, `packages/eusolicit-kraftdata/`.
- Add `--remove-orphans` (or explicit old-container removal) to `scripts/deploy.sh` so the
  `ai-gateway → *-gateway` cutover stops colliding on port 18004 (root cause of the 2026-05-22
  failed deploy/rollback).
- **Risk:** Low. **Payoff:** High — removes the rename drift that broke today's deploy.

### P1 — Docs rename *(no code risk)*
- Tier 1 authoritative docs (tracked edits, architecture review): PRD + amendments, architecture +
  amendments. Re-issue the May-12 amendments under `-agenticsai` filenames.
- Tier 2/3 bulk replace across epics, implementation-artifacts, runbooks, test artifacts; rename
  files containing legacy tokens. Update CLAUDE.md / GEMINI.md guidance.
- **Risk:** Low. Keep git history; spot-check Tier 1 for grammatical coherence.

### P2 — Code rename, no DB *(coordinate with orchestrator)*
- `packages/eusolicit-sirmaai` → `eusolicit-agenticsai` (import `eusolicit_agenticsai`); update all
  consumers in one commit (no compat shim, per the S04.30 precedent).
- `sirmaai-gateway` → `agenticsai-gateway` (service, module `agenticsai_gateway`, classes
  `AgenticSAI*`, Dockerfile, compose, Makefile, Helm, CI matrix).
- Env vars `SIRMAAI_*`/`AIGW_*` → `AGENTICSAI_*` in settings classes (prod `.env` updated in P4's window).
- **Risk:** Medium — large mechanical change; lockstep across services + CI. Pause orchestrator.

### P3 — Optionality + provider interface
- Introduce `AIProvider` ABC + `AgenticSAIProvider` + `NoopProvider`; `AI_PROVIDER` config.
- No-AI mode: non-fatal startup (gate `init_registry`/rate-limits/Fernet behind provider),
  consumer short-circuit, feature hiding.
- Rename provider-specific vocabulary (`kraftdata_id`, `call_kraftdata`, `n8n_subdomain`) to neutral
  terms at the boundary.
- **Risk:** Medium — behavioural; needs tests for no-AI mode and provider selection.

### P4 — Isolation + DB rename *(highest risk; maintenance window)*
- Move tenant state `client.sirmaai_projects` → `agenticsai.projects`; replace cross-schema access
  with gateway API; drop/formalize the cross-schema FK.
- Rename `gateway` schema → `agenticsai`; `ai_gateway_role` → `agenticsai_role`; columns
  (`sirmaai_project_id` → `agenticsai_project_id`); Celery task names.
- Update infra init scripts (`infra/`), Alembic migrations across services, prod `.env` on www1.
- **Risk:** High — Alembic migrations + role rename + prod DB downtime. Stage on a DB clone; have a
  rollback migration; schedule a maintenance window. Note role rename touches `DATABASE_URL`
  connection strings and RBAC tests (~126 refs).

---

## 6. Risks & coordination notes

- **Orchestrator contention:** the pipeline autonomously edits/commits `eusolicit-app` on `main`.
  Code phases (P0, P2, P3, P4) must pause it (`kill -TERM <pid>` → restart `start-orchestrator.sh
  --skip-bootstrap`) or run on a coordinated branch; otherwise auto-sync commits will collide with
  the rename.
- **CI is billing-dark:** GitHub Actions is org-blocked, so PRs can't be CI-verified; merges have
  been admin-merges. Renames of this scale are exactly where un-verified drift hides — restore CI
  billing before P2+ if at all possible, or gate each phase behind local `make lint/type-check` +
  the new `make migrate-check`.
- **Prod surface drift:** prod `.env`, nginx, and compose on www1 are host-only state not in the
  repo. Env-var and service renames (P2/P4) require coordinated www1 changes; otherwise prod breaks
  on next deploy (see the 2026-05-22 incident).
- **Migration history is immutable:** existing Alembic files keep their old identifiers in
  historical revisions; the rename is a *new* forward migration, not an edit of past ones.

---

## 7. Open items for sign-off

- Approve the phased sequencing and the P4 maintenance-window requirement.
- Confirm env-var naming target (`AGENTICSAI_*` vs a shorter `AGS_*`) before P2.
- Decide whether P1 (docs) proceeds in parallel with P0 (it has no code risk) or waits.
- Confirm Helm/PagerDuty/observability rename scope (service name appears in alerting/escalation).

---

## 8. Revision 2026-05-23 — reviewed against the AgenticSAI integration guide

Re-reviewed the gaps and approach against the authoritative platform contract
(`agenticsai.endigitalx.com APIs/AGENTICSAI_API_INTEGRATION_GUIDE.md`, OpenAPI 3.1.0,
v1.0). The guide **validates the overall direction** and sharpens four points.

### 8.1 Canonical host (directive)
`https://agenticsai.endigitalx.com` is the **single canonical base URL** going forward
(guide §2). `stage.sirma.ai` and `kraftdata.ai` are **obsolete** — treat any reference as
stale.
- **Code (P2/P3):** the gateway's outbound defaults `sirmaai_base_url` *and*
  `kraftdata_base_url` (`services/sirmaai-gateway/src/sirmaai_gateway/config.py:46,79`) still
  default to `https://stage.sirma.ai` → set both to `https://agenticsai.endigitalx.com`
  (the data-pipeline n8n template tests already enforce this host). Update test fixtures off
  `stage.sirma.ai`. This is the **external** AgenticSAI host, distinct from the *internal*
  consumer→gateway URL (the `ai-gateway:8000` drift handled separately).
- **Docs (P1):** living docs updated to `agenticsai.endigitalx.com`.

### 8.2 Validated — the integration already follows the recommended pattern
- The gateway calls the **Public Integration API** `/client/api/v1/...` with project-scoped
  **`X-API-Key`** (verified: `sirmaai_key_client`, `sirmaai_inventory_client`,
  `sirmaai_async_client`, `execution.py`), exactly the guide's §3.1/§10.1 M2M recommendation
  — **not** the first-party `/api/...` console surface. No change needed; **not a gap**.
- **Tenancy = project-per-tenant** (one Org, one Project per company, per-Project key) matches
  guide §10.5's "hard isolation" recommendation — the correct model for the strict-isolation
  goal. Keep.
- Per-request degradation (503 → `AGENT_UNAVAILABLE`), circuit breaker, reconciler, Standard
  Webhooks receiver all align with guide §6/§8/§10.6.

### 8.3 Sharpened gaps
1. **`agents.yaml` fail-fast is doubly wrong.** Beyond the availability bug (the 2026-05-22
   prod crash), the guide explicitly says *discover capabilities dynamically* (list
   agents/teams/workflows; `GET /api/webhooks/event-types` "call this first; do not hard-code
   the catalogue", §8/§10.1). The architecture already plans to retire `agents.yaml` for the
   dynamic resolver (`SirmaAIAgentResolver`, on-demand inventory re-sync). **P3 should finish
   that migration and make startup non-fatal** (discover + cache with TTL per §10.6), removing
   the static-registry dependency entirely rather than just guarding it.
2. **Scope the provider abstraction to EU Solicit's *usage*, not AgenticSAI's surface.**
   AgenticSAI is a rich platform (Agents/Teams/Workflows, Storage Resources, MCP, Policies,
   Voice, Traces, Standard Webhooks). A drop-in clone is unrealistic, so the `AIProvider`
   interface (D1 "swappable") must be defined around what EU Solicit actually consumes —
   **agent run (sync/SSE/async-poll), KB upload+semantic-search, webhook events, and tenant
   Project provisioning**. A second provider then satisfies that narrower contract (possibly by
   composing several services or a self-hosted stack), which makes "swappable" realistic.
3. **Credential boundary nuance for isolation.** Webhook *subscription management* uses an
   operator-issued **`bearer-token`** (guide §8), higher-privilege than the per-project
   `X-API-Key`. The gateway holds this operator credential — it must be vaulted gateway-side
   and never exposed to domain services. Add to the P4 isolation/credential review.
4. **Idempotency on run submission.** Guide §4.3: `POST` runs are not inherently idempotent —
   the async-run path should carry a client-side dedupe/correlation key. Confirm
   `submit_agent_run_async` does (P3 hardening).

### 8.4 Net effect on the plan
No phase is added or removed. **P2** also retargets the external base-URL defaults to
`agenticsai.endigitalx.com`. **P3** absorbs: finishing the dynamic-resolver migration +
non-fatal startup (retiring `agents.yaml`), scoping the `AIProvider` interface to EU Solicit's
usage, and the idempotency-key hardening. **P4** adds the operator-bearer-token vaulting to the
credential/isolation review. The strict-isolation and optional+swappable goals remain sound and
are, if anything, better-supported now that the contract is confirmed to be a clean, versioned,
project-scoped REST surface.
