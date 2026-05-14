# Story 4.31: Local-dev SirmaAI Configuration

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer landing the last gap in the SirmaAI pivot's local-dev story (Concern #8 from `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-12-sirmaai.md` lines 313–319 — "ADR-019 *Consequences* names two options (point dev at staging vs build a `sirmaai-mock` thin FastAPI). Neither commits in an epic story… First dev who tries `make up` for the first time will discover the gap.") and the explicit injection point in Epic 4 amendment line 513 (`S04.31 Local-dev SirmaAI configuration | 2 | backend + docs | For `make up` local-dev: point at SirmaAI staging (`stage.sirma.ai` or shared dev Org on `agenticsai.endigitalx.com`) via env override `SIRMAAI_BASE_URL` + dev Org/Project credentials in `.env.example`. Shared dev Project allocated outside the per-tenant flow (admin-bootstrapped). Documentation in `eusolicit-app/CLAUDE.md` under "Local stack". `sirmaai-mock` deferred to when test-isolation pain warrants (per ADR-019). Closes readiness Concern #8.`)**,

I want **a small, docs-and-config-only landing that (a) adds a canonical `sirmaai_base_url` field to `services/sirmaai-gateway/src/sirmaai_gateway/config.py` (read from the `SIRMAAI_BASE_URL` env var via `SIRMAAI_GATEWAY_` env prefix → effective env key `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` per the pydantic-settings convention used everywhere else in the file; defaulting to `"https://stage.sirma.ai"` to match ADR-019's "Option 1: point local dev at SirmaAI staging" recommendation and the existing legacy `kraftdata_base_url` default which was the same value pre-pivot), WITHOUT renaming or removing the existing `kraftdata_base_url` field (per the project `CLAUDE.md` "Active Service Migrations" rule — "Consumer base-URL configs (`CLIENT_API_AIGW_BASE_URL`, `ADMIN_API_AIGW_BASE_URL`, `AI_GATEWAY_BASE_URL`) are migrated by later stories; do not rename them in this story"; the same discipline applies to `kraftdata_base_url` → `sirmaai_base_url` for the in-process gateway-local field — the rename is a separate hardening story, this story only ADDS the canonical name); (b) un-comments and documents the existing commented-out `SIRMAAI_GATEWAY_ENABLED`, `SIRMAAI_ORG_ID`, `SIRMAAI_FERNET_KEY`, `PROJECT_CACHE_TTL_SECONDS` block in `eusolicit-app/.env.example` (lines 122–142) so a fresh `cp .env.example .env` produces a working local-dev `.env` that boots the SirmaAI code path on `make up` rather than landing on the pre-amendment KraftData path (flag-off remains the safe per-env default — the un-commenting documents the SHAPE of the variables; their effective values for staging-dev are provided in `eusolicit-app/CLAUDE.md` per AC 4); (c) injects the missing `SIRMAAI_BASE_URL` env var entry into the same block — currently absent from `.env.example` (verified by grep) even though it is referenced by ADR-019 and Epic 4 amendment line 513 as the canonical override key; (d) adds explicit `.env.example` entries for the four operator-supplied dev-bootstrap credentials a fresh dev needs to actually call SirmaAI staging successfully: `SIRMAAI_ADMIN_API_KEY` (the Org-level admin key used by `S04.22` rotation — without it the rotation Celery worker imports OK but cannot run; documented but kept empty in the example, with the comment pointing at the runbook for how to obtain the shared-dev-Org key), `SIRMAAI_WEBHOOK_CALLBACK_URL` (the public URL where SirmaAI delivers webhooks — for local-dev typically a `ngrok` or `cloudflared` tunnel to `localhost:8004/webhooks/sirmaai` per S04.25, kept empty in the example with a `# Local dev: see eusolicit-app/CLAUDE.md → Local stack → SirmaAI webhooks for tunnel setup` comment), and the shared-dev Org/Project bootstrap pointers (`SIRMAAI_ORG_ID` = the shared dev Org UUID, kept empty with a `# Ask Deb for the shared-dev Org id` comment per the admin-bootstrap pattern; the per-Project `api_key_encrypted` does NOT go in `.env.example` because per-Project keys are stored in `client.sirmaai_projects` Fernet-encrypted by S04.21 — they are bootstrapped through `admin-api` not env vars, per the "Shared dev Project allocated outside the per-tenant flow (admin-bootstrapped)" amendment line); (e) creates a new "Local stack" section in `eusolicit-app/CLAUDE.md` — the file does NOT currently exist (verified — only the project-root `/home/debian/Projects/eusolicit/CLAUDE.md` exists), and the S04.20 dev-story retrospective at `eusolicit-docs/implementation-artifacts/4-20-service-rename-and-flag-scaffold.md` LOW finding L-1 (line 474) flagged "`eusolicit-app/CLAUDE.md` referenced by Task 10.1 does not exist; only the project-level `/home/debian/Projects/eusolicit/CLAUDE.md` exists. The dev correctly updated the project-level one… The story's task text should be corrected post-hoc to point to the actual file, or the project should create the per-app CLAUDE.md if that was the intent." — THIS story chooses the second option per Epic 4 amendment line 513's explicit `eusolicit-app/CLAUDE.md` reference, by creating the per-app file with a focused scope: an "Important note" pointer to the project-root CLAUDE.md as the canonical reference for global rules, a "Local stack" section explaining `make up` SirmaAI integration, a "SirmaAI dev credentials" subsection explaining the admin-bootstrap path for a shared dev Org/Project, and an "Optional: ngrok tunnel for SirmaAI webhooks" subsection covering the local-dev webhook-receive path; the per-app file remains a focused supplement, NOT a fork of the global file — to avoid the drift the S04.20 retro implicitly warned about; (f) adds a new runbook at `eusolicit-docs/runbooks/sirmaai-local-dev.md` covering the admin-bootstrap procedure for obtaining the shared-dev Org/Project credentials, the `.env` configuration steps a fresh dev executes verbatim, the `make up` smoke check, and the failure-mode pointers (what error message the dev sees when each credential is missing) — the runbook is the dev-friendly counterpart to the AC-level docs in `eusolicit-app/CLAUDE.md` and is referenced from there; (g) explicitly DOES NOT build the `sirmaai-mock` thin-FastAPI stub — per ADR-019 line 93 ("(1) for early E24/E25/E26 work, (2) later when test-isolation pain forces it. Don't pre-build the mock.") and Epic 4 amendment line 513 ("`sirmaai-mock` deferred to when test-isolation pain warrants (per ADR-019)") — local-dev points at SirmaAI staging via the shared-dev Org for v1, and the mock is a future story when the integration-test isolation pain warrants it (likely E26 or later); (h) explicitly DOES NOT touch `MOCK_KRAFTDATA_URL=http://localhost:9999` at `.env.example` line 26 — that env var serves a different purpose (it is consumed by `respx`-driven integration tests as the URL `httpx` mocks intercept, NOT a real backend the gateway points at), and removing it would break the test-suite mocking layer; the var is preserved verbatim, with a documentation hint clarifying the distinction added as a `# Used by respx integration test mocks; NOT a real backend — see .env.example below for SIRMAAI_BASE_URL` comment**,

so that **(1) Concern #8 from `implementation-readiness-report-2026-05-12-sirmaai.md` (line 313) and the summary row at line 426 flips from ❌ GAP to ✅ Covered — a fresh backend developer cloning `eusolicit/` and running `make up` for the first time has a clear, single-page-with-runbook path to a working SirmaAI integration via the shared dev Org, instead of "first dev who tries `make up` discovers the gap"; (2) Epic 4 amendment line 513's S04.31 description becomes implementation rather than aspiration — the four enumerated deliverables (a `SIRMAAI_BASE_URL` env override, dev Org/Project credentials shape in `.env.example`, admin-bootstrap of the shared dev Project, and `eusolicit-app/CLAUDE.md` Local stack docs) all land in one commit with their cross-references intact; (3) ADR-019's "Local-dev story is uglier" footnote at architecture-amendment line 93 is closed for Option 1 (point at staging) without committing to Option 2 (the mock) — the door stays open for the mock when E26 integration tests start fighting against shared-staging-Org isolation, but no premature work is done now (per memory `feedback_investigate_dont_quiz.md` + the Epic 4 amendment "Don't pre-build the mock" guidance); (4) the SirmaAI pivot's first-dev-experience invariant is honoured — the next dev who joins the team after the pivot lands has a self-serve onboarding path that does NOT require pinging the engineering channel to figure out which env vars to set, which is the consistent friction point in pre-pivot SirmaAI-related dev work (per the implementation-readiness report's risk model where "developer productivity" is listed as the explicit impact of Concern #8); (5) the project-level `/home/debian/Projects/eusolicit/CLAUDE.md` is NOT modified by this story (it already has the `### Active Service Migrations` section at line 77 with the S04.20 + S04.22 + S04.23 + S04.24 + S04.25 + S04.26 + S04.30 entries — adding an S04.31 entry there would duplicate the per-app file's "Local stack" section; instead this story adds the per-app `eusolicit-app/CLAUDE.md` with a "see project-root CLAUDE.md for global rules" pointer, and the project-root CLAUDE.md's existing `## Repository Layout` block which already points readers at `eusolicit-app/` for code work is the discovery hook); (6) the package-level rename closure landed by S04.30 (`eusolicit-kraftdata` → `eusolicit-sirmaai`, 2026-05-14) chains correctly into the local-dev story — the dev who just `pip install -e packages/eusolicit-sirmaai/` now has a matching `.env.example` shape that points at the right SirmaAI host, instead of an inconsistent "package renamed but env vars still say KraftData" middle state; (7) the per-Project key-vault foundation landed by S04.22 (`sirmaai_rotate_project_keys` Celery Beat at line 81 of project CLAUDE.md) gets its corresponding dev-side bootstrap docs — without S04.31 a fresh dev cannot get a per-Project `api_key_encrypted` populated to actually exercise S04.22's rotation flow locally, which would silently hide rotation bugs until staging integration testing; (8) the change set is minimal and reversible — no business logic, no schema changes, no Docker image changes, no Helm chart changes; one config-file edit (`config.py` adds one field with a default), one `.env.example` block expansion (un-comment and document existing variables + add the missing `SIRMAAI_BASE_URL`), one new file (`eusolicit-app/CLAUDE.md`, ~50 lines), and one new runbook (`eusolicit-docs/runbooks/sirmaai-local-dev.md`, ~70 lines). Rollback is a single `git revert` — no operator action needed; (9) the AP18-C2 atomic two-gate landing pattern (used by S04.27/S04.28/S04.29/S04.30) is honoured — the story file + sprint-row flip + the actual code+docs commit all land together; the dev pass MUST land everything in one commit so a partial-rollback never produces a state where `eusolicit-app/CLAUDE.md` exists but its referenced runbook does not (or vice versa)**.

## Acceptance Criteria

1. **Canonical `SIRMAAI_BASE_URL` field added to gateway config** — `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/config.py`:
   - A new field `sirmaai_base_url: str = "https://stage.sirma.ai"` is appended to the `SirmaAIGatewaySettings` class, placed in the "SirmaAI tenant mapping settings (S04.21)" block at line 70–85 (just above the `sirmaai_org_id` field) since it is conceptually a tenant-mapping prerequisite — every tenant Project lookup needs the base URL.
   - The field's docstring states: *"Base URL of the SirmaAI agentic platform. Defaults to ADR-019 staging (`stage.sirma.ai`) for local-dev per S04.31. Override per-environment via the `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` env var. Production deployments point at `https://agenticsai.endigitalx.com/` per memory `reference_sirmaai_api_docs.md`. NEVER hard-code per-environment URLs in business logic — always read via `get_settings().sirmaai_base_url`."*
   - The existing `kraftdata_base_url: str = "https://stage.sirma.ai"` field at line 46 is **preserved verbatim** — no rename, no removal. Per project `CLAUDE.md` "Active Service Migrations" rule ("Consumer base-URL configs… are migrated by later stories; do not rename them in this story"), the legacy field continues to coexist with the canonical one while the SirmaAI cutover is in progress. Field-rename retirement is scheduled as a separate hardening story.
   - The `sirmaai_base_url` field's env-var name resolves through pydantic-settings to `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` (because `BaseServiceSettings` sets `env_prefix="SIRMAAI_GATEWAY_"` per the per-service prefix convention listed in the project-level delivery instructions: `"CLIENT_API_, ADMIN_API_, DATA_PIPELINE_, AI_GATEWAY_, NOTIFICATION_, INTEGRATIONS_API_"` and the analog for sirmaai-gateway). **Pre-flight verify** in the dev pass that the actual prefix used by `BaseServiceSettings` for `sirmaai-gateway` is `SIRMAAI_GATEWAY_` (NOT just `SIRMAAI_`) by inspecting the existing `webhook_replay_window_seconds: int = 300  # env: SIRMAAI_GATEWAY_WEBHOOK_REPLAY_WINDOW_SECONDS` comment at line 113 — that comment is the source-of-truth for the prefix. If the prefix is different (e.g. just `SIRMAAI_`), document the prefix in the field docstring AND in `.env.example` AND in `eusolicit-app/CLAUDE.md`.
   - No callers are updated in this story — the field is added but no code reads it yet. The S04.23 `SirmaAIAgentResolver` and the S04.24 `submit_agent_run_async` paths continue to use the existing `kraftdata_base_url`. Switching readers from `kraftdata_base_url` to `sirmaai_base_url` is **out of scope** — separate hardening story.

2. **`SIRMAAI_BASE_URL` env-var entry added to `eusolicit-app/.env.example`** — the SirmaAI Gateway block at lines 122–142:
   - A new commented-out line `# SIRMAAI_GATEWAY_SIRMAAI_BASE_URL=https://stage.sirma.ai` is added immediately AFTER the `# SIRMAAI_GATEWAY_ENABLED=false` line at line 128, before the `# SIRMAAI_ORG_ID=` line at line 132.
   - The accompanying comment block (kept short — `.env.example` favours brief explanations) states: *"# SirmaAI agentic platform base URL. Defaults to ADR-019 staging (`stage.sirma.ai`).\n# Override for prod to `https://agenticsai.endigitalx.com/` per memory `reference_sirmaai_api_docs.md`.\n# Local-dev: leave commented to use the in-code default."*
   - The variable name in `.env.example` matches the pydantic-settings env-key derived from AC 1's `env_prefix` resolution (i.e. `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` if the prefix is `SIRMAAI_GATEWAY_`; otherwise the bare `SIRMAAI_BASE_URL`). The dev pass MUST verify the exact key at pre-flight grep and use the verified name verbatim — a typo here is invisible at boot time (pydantic-settings silently ignores unknown env vars per `extra="ignore"` at line 40 of `config.py`).

3. **`SIRMAAI_ADMIN_API_KEY` + `SIRMAAI_WEBHOOK_CALLBACK_URL` env-var entries added to `.env.example`**:
   - Both are added to the same SirmaAI Gateway block, AFTER the `# PROJECT_CACHE_TTL_SECONDS=300` line at line 141 (the current end of the block).
   - `# SIRMAAI_ADMIN_API_KEY=` — comment: *"# Org-level admin API key for SirmaAI per-Project key creation + revocation (S04.22).\n# REQUIRED when SIRMAAI_GATEWAY_ENABLED=true AND the rotation worker is running.\n# Local-dev: ask the project lead for the shared-dev Org admin key.\n# See eusolicit-docs/runbooks/sirmaai-local-dev.md for the bootstrap procedure.\n# SECURITY: never commit a real value — keep this commented in .env.example."*
   - `# SIRMAAI_WEBHOOK_CALLBACK_URL=` — comment: *"# Public URL where SirmaAI delivers webhooks (S04.25). Bootstrap refuses to run if empty.\n# Local-dev: use a tunneling proxy (ngrok or cloudflared) to expose localhost:8004/webhooks/sirmaai.\n# Prod: routed via the public ingress at https://api.eusolicit.com/webhooks/sirmaai per S04.29.\n# See eusolicit-app/CLAUDE.md → Local stack → SirmaAI webhooks for tunnel setup."*
   - Both lines remain commented in `.env.example` — the file is templated, not the runtime config. A dev's actual `.env` is uncommented at copy-time per their environment.

4. **`eusolicit-app/CLAUDE.md` created** — the file does NOT currently exist (per S04.20 dev-story retro L-1 finding at line 474 of `4-20-service-rename-and-flag-scaffold.md`); this story creates it with a focused scope (~40–60 lines) covering:
   - **Header section** (first 5 lines): `# eusolicit-app — CLAUDE.md` + a "see also" pointer: *"For project-wide CLAUDE.md rules (architecture, commands, service ports, RBAC, testing strategy, critical patterns), see `/home/debian/Projects/eusolicit/CLAUDE.md` at the repo root. This file is the per-app supplement — currently scoped to the local-stack SirmaAI integration. New per-app rules MUST be added here, not the root CLAUDE.md, to prevent drift."*
   - **`## Local stack` section** with subsections:
     - `### Quick start` — verbatim copy of the project-root CLAUDE.md `make infra` / `make up` block (lines 28–31 of root CLAUDE.md), with a one-line "what to expect" pointer to the rest of this section.
     - `### SirmaAI integration` — explains: (a) `make up` boots the `sirmaai-gateway` service on port 8004 (per the project-root CLAUDE.md Service Ports table); (b) the gateway defaults to SirmaAI staging at `https://stage.sirma.ai` per ADR-019 + S04.31; (c) the feature flag `SIRMAAI_GATEWAY_ENABLED=false` is the default — flip it to `true` in `.env` to exercise the SirmaAI code path locally; (d) the legacy KraftData path remains the runtime behaviour while the flag is off (per S04.20 cutover discipline); (e) the canonical override env var is `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` (or the bare `SIRMAAI_BASE_URL` if the AC-1 prefix verification lands on the bare form — the doc reflects whatever the pre-flight grep confirms).
     - `### SirmaAI dev credentials` — explains the admin-bootstrap path: (a) per-Project API keys (`api_key_encrypted` in `client.sirmaai_projects`) are NOT in `.env` — they are bootstrapped through `admin-api` per the "Shared dev Project allocated outside the per-tenant flow" amendment line, with a procedure-link to `eusolicit-docs/runbooks/sirmaai-local-dev.md`; (b) the `SIRMAAI_GATEWAY_SIRMAAI_ADMIN_API_KEY` env var (Org-level admin key for rotation) requires the shared-dev Org admin key — *"Ask Deb"* with a one-line context for why this is operator-handed-out not committed; (c) the `SIRMAAI_GATEWAY_SIRMAAI_FERNET_KEY` is locally-generated per the `.env.example` comment at line 138 (`python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`) — same pattern as `CALENDAR_ENCRYPTION_KEY` for E09.
     - `### Optional: ngrok tunnel for SirmaAI webhooks` — short subsection explaining: (a) `make up` does not expose `localhost:8004/webhooks/sirmaai` to the public internet, so SirmaAI staging cannot deliver webhooks back; (b) the recommended local-dev pattern is `ngrok http 8004` (or `cloudflared tunnel run`) to get a public HTTPS URL; (c) set `SIRMAAI_GATEWAY_SIRMAAI_WEBHOOK_CALLBACK_URL=https://<your-ngrok>.ngrok.io/webhooks/sirmaai` in `.env`; (d) restart the gateway (`docker compose restart sirmaai-gateway`); (e) bootstrap a webhook subscription via the documented S04.25 path; (f) the tunnel is local-dev-only — prod uses the public ingress at `api.eusolicit.com/webhooks/sirmaai` per S04.29; (g) `sirmaai-mock` is NOT yet built (per ADR-019 — deferred to E26+) so this is currently the only end-to-end local-dev webhook-receive path.
   - **Footer line** at the end: *"Last updated: 2026-05-14 by S04.31 (eusolicit-docs/implementation-artifacts/4-31-local-dev-sirmaai-configuration.md)"* — gives the next-toucher a self-anchoring update marker.
   - Length cap: aim for ≤80 lines. The per-app file is a focused supplement, NOT a fork. Anything that wants to grow past 80 lines belongs in the project-root CLAUDE.md or a dedicated runbook.

5. **`eusolicit-docs/runbooks/sirmaai-local-dev.md` created** — a new runbook (~70 lines) covering the operator-facing detailed steps that don't fit in the AC-4 inline docs:
   - **Header**: `# Runbook: SirmaAI Local-Dev Configuration` + status row (last-updated, owner = backend, related stories = S04.21/S04.22/S04.25/S04.31).
   - **`## Purpose`** — one paragraph explaining the runbook's scope: "A fresh dev cloning `eusolicit/` for the first time gets a working `make up` against SirmaAI staging within 15 minutes. Out of scope: production SirmaAI deployment (see `eusolicit-docs/runbooks/sirmaai-key-rotation.md` + `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`)."
   - **`## Pre-flight check`** — assert: docker is running, `eusolicit/` is cloned, `.env.example` exists (the canonical template).
   - **`## Step-by-step`** numbered list:
     1. `cp eusolicit-app/.env.example eusolicit-app/.env` — fresh dev copy.
     2. Generate a local Fernet key: `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`; paste the value into the `SIRMAAI_GATEWAY_SIRMAAI_FERNET_KEY=` line in `.env`, uncommenting it.
     3. Ask the project lead (Deb) for the **shared-dev Org id** and the **shared-dev Org admin API key**. Paste them into `SIRMAAI_GATEWAY_SIRMAAI_ORG_ID=...` and `SIRMAAI_GATEWAY_SIRMAAI_ADMIN_API_KEY=...` in `.env`. (Document: these are NOT committed because the shared-dev Org is shared across all developers and rotating it requires a coordinated re-bootstrap of every dev's `.env` — operator-hand-off pattern, same as the Stripe test-mode key pattern for Story 15.0.)
     4. Set `SIRMAAI_GATEWAY_ENABLED=true` to exercise the SirmaAI code path locally. (Note: leave it commented to use the pre-amendment KraftData path during the cutover window.)
     5. (Optional, only if you need webhook receive) Set up an `ngrok` tunnel: `ngrok http 8004`. Copy the public HTTPS URL. Set `SIRMAAI_GATEWAY_SIRMAAI_WEBHOOK_CALLBACK_URL=https://<your-ngrok>.ngrok.io/webhooks/sirmaai` in `.env`.
     6. `cd eusolicit-app && make infra && make migrate-all && make up` — boots the full stack with SirmaAI integration enabled.
     7. **Smoke check**: `curl http://localhost:8004/health` → `{"status":"ok"}`; `curl http://localhost:8004/ready` → `200 OK`. (Document expected output verbatim so the dev can pattern-match.)
     8. **Admin-bootstrap the shared dev Project** — call `admin-api`'s tenant-provisioning endpoint to populate `client.sirmaai_projects` with the shared dev Org+Project mapping. Exact call shape: `POST http://localhost:8002/admin/sirmaai-projects/bootstrap` with `{ "company_id": "<test-company-uuid>", "sirmaai_org_id": "<from-step-3>", "sirmaai_project_id": "<shared-dev-project-uuid>" }` (per S04.21's admin-api integration; the dev pass MUST verify this exact endpoint shape against the current `admin-api` route table before committing — the runbook's value depends on the example being copy-paste-runnable).
   - **`## Failure modes`** — table mapping error messages to root causes:
     | Symptom | Root cause | Fix |
     |---|---|---|
     | `make up` fails with `pydantic.ValidationError: sirmaai_fernet_key` required | `SIRMAAI_GATEWAY_SIRMAAI_FERNET_KEY` unset while `SIRMAAI_GATEWAY_ENABLED=true` | Run step 2 above |
     | `make up` succeeds but `/admin/circuits` 500s on first call | `SIRMAAI_GATEWAY_SIRMAAI_ORG_ID` or admin key unset | Run step 3 above |
     | Webhook bootstrap fails with `WEBHOOK_BOOTSTRAP_ENABLED but callback URL empty` | `SIRMAAI_GATEWAY_SIRMAAI_WEBHOOK_CALLBACK_URL` unset while bootstrap is enabled | Run step 5 above |
     | Calls to SirmaAI hang for 60s+ and return 504 | Default base URL `stage.sirma.ai` is unreachable from local network | Verify staging is up; consider VPN; check ADR-019 for current canonical staging URL |
   - **`## Rollback`** — single line: *"Comment out the SirmaAI block in `.env` and restart the stack. The flag-off path (legacy KraftData behaviour per S04.20) is preserved as the runtime fallback."*
   - **`## Related`** — link list pointing at: project-root CLAUDE.md, eusolicit-app/CLAUDE.md, S04.21–S04.25 story files, ADR-019 in `architecture-amendment-2026-05-12-sirmaai.md`, and the deferred `sirmaai-mock` follow-up referenced in ADR-019.

6. **`MOCK_KRAFTDATA_URL` env-var preservation** — `.env.example` line 26 (`MOCK_KRAFTDATA_URL=http://localhost:9999`):
   - This line is **preserved verbatim** — no rename, no removal. The variable is consumed by `respx`-driven integration tests as the URL `httpx` mocks intercept; renaming it would break the test layer.
   - A short hint comment is added immediately above the line: `# Mock backend URL for respx integration tests; NOT a real backend. See SIRMAAI_GATEWAY_SIRMAAI_BASE_URL below for the real-backend override.`
   - The dev pass MUST verify by grepping `MOCK_KRAFTDATA_URL` across `eusolicit-app/tests/` AND `eusolicit-app/services/sirmaai-gateway/tests/` that the variable is actually consumed (not orphan). If grep returns zero hits, document in Completion Notes that the variable can be removed in a follow-up story — but do NOT remove it in this story (out of scope).

7. **No `sirmaai-mock` stub built** — per Epic 4 amendment line 513 + ADR-019 line 93 ("Don't pre-build the mock"):
   - No new `services/sirmaai-mock/` directory.
   - No new mock FastAPI stub binary.
   - No `docker-compose.yml` entry for a mock service.
   - The deferral is documented in `eusolicit-app/CLAUDE.md` (AC 4 → `### Optional: ngrok tunnel for SirmaAI webhooks` subsection mentions "sirmaai-mock is NOT yet built") and in the runbook's `## Related` section (AC 5 → link to ADR-019 footnote + a forward-pointer to "E26+ integration tests when test-isolation pain forces the mock").

8. **Files this story touches (definitive list)** — used by the dev pass as a checklist; any file outside this list must be a justified addition documented in Completion Notes:
   - `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/config.py` (one new field per AC 1).
   - `eusolicit-app/.env.example` (three new lines per AC 2 + AC 3 + one hint comment per AC 6).
   - `eusolicit-app/CLAUDE.md` (new file, ~40–80 lines per AC 4).
   - `eusolicit-docs/runbooks/sirmaai-local-dev.md` (new file, ~50–80 lines per AC 5).
   - `eusolicit-docs/implementation-artifacts/sprint-status.yaml` (one row flipped backlog → ready-for-dev for `4-31-local-dev-sirmaai-configuration` — this is the bmad-create-story dispatch; the dev pass flips ready-for-dev → review/done as it lands).
   - **NOT** the project-root `/home/debian/Projects/eusolicit/CLAUDE.md` — this story does NOT modify it. The "Active Service Migrations" section at line 77 already covers S04.20 + S04.22 + S04.23 + S04.24 + S04.25 + S04.26 + S04.30; S04.31 is intentionally NOT added there to avoid duplicating the per-app `eusolicit-app/CLAUDE.md` Local stack section. If a future operator wants a one-line crumb at root-CLAUDE.md, that is a separate optional doc-polish landing.

9. **No regression — every existing test stays green**:
   - `make lint` (`ruff check services/ packages/ tests/`) clean — the new `sirmaai_base_url` field adheres to the existing snake_case + type-hint conventions used by every other field in `config.py`.
   - `make type-check` (`mypy services/ packages/`) clean — the new field has an explicit `str` annotation and a default value, satisfying mypy's strict-optional rules.
   - `make test-unit` clean — no test imports the renamed field by name; the addition is purely additive.
   - `make test-service SVC=sirmaai-gateway` clean — the existing `SirmaAIGatewaySettings` instantiation in `get_settings()` continues to work; the new field's default value means no env var is required at test-time.
   - `make test-integration` clean (when `make infra` is running) — no integration test asserts on the absence of `sirmaai_base_url`.
   - The pre-existing `kraftdata_base_url` field continues to function exactly as before — no caller is rewired.
   - **Per memory `project_test_execution_environment.md`**: if the dev pass runs in a host venv without provisioned service deps, the dev pass falls back to ruff + collection-only + syntax checks at the host level and relies on CI for full runtime validation. Document the chosen verification path in Completion Notes.

10. **DoD per project delivery rules** (the relevant subset — this is a docs + config story, not a behavioural change):
    - `make lint` passes.
    - `make type-check` passes.
    - `make test-unit` passes (no new tests required — the addition is purely additive and the existing scaffold-level field-presence tests, if any, continue to pass).
    - `make test-integration` passes (assumes `make infra` running — see AC 9 fallback).
    - `make coverage` passes (≥80% line coverage on touched modules — `config.py` coverage is unchanged because the new field has a default and no code reads it yet).
    - Frontend gates — N/A (no frontend code touched).
    - E2E gates — N/A (no user flow touched).
    - **NEW manual verification step** unique to this docs story: the dev pass MUST `cp eusolicit-app/.env.example /tmp/test-env-from-example` then assert that the diff between `/tmp/test-env-from-example` and the pre-story `.env.example` matches the AC 2 + AC 3 + AC 6 expected additions exactly (no stray edits, no whitespace drift). Document the diff inspection in Completion Notes.

11. **No business logic, no schema migration, no Docker image change** — explicit invariants:
    - No Alembic migration created.
    - No Docker image rebuild required (the new config field has a default, so existing baked images continue to start without env overrides).
    - No Helm chart change (`infra/helm/` untouched).
    - No CI workflow change (`.github/workflows/` untouched).
    - No `pyproject.toml` change in any service or package.
    - The story is purely a docs + single-line-config landing.

12. **Cross-tenant negative test invariant** (project memory rule, explicitly called out even though this story is non-business-logic):
    - The local-dev config does not introduce any new cross-tenant code paths. The shared-dev Org+Project pattern is intentionally non-multi-tenant — it is the local-dev-only shared workspace, NOT a production multi-tenant pattern.
    - The runbook (AC 5 step 8) explicitly notes that admin-bootstrapping the shared dev Project bypasses the production per-tenant flow on purpose, and this bypass is local-dev-only (the production `admin-api` route must NOT expose a `POST /admin/sirmaai-projects/bootstrap` endpoint that allows arbitrary Project mapping — production per-tenant provisioning lives in E24, which has its own cross-tenant defenses).
    - Document in Completion Notes: "AC 12 — no cross-tenant code paths touched; the local-dev shared-dev Project pattern is documented as local-dev-only with an explicit forward-pointer to E24's production provisioning."

13. **Sprint-status row flip** — `eusolicit-docs/implementation-artifacts/sprint-status.yaml`:
    - The `4-31-local-dev-sirmaai-configuration` row at line 445 transitions `backlog` → `ready-for-dev` as part of the bmad-create-story dispatch (this dispatch). The dev pass later transitions `ready-for-dev` → `review` (after landing) → `done` (after code-review).
    - The `last_updated` field at the top of the file MUST be updated to `2026-05-14` with a bmad-create-story crumb mentioning S04.31.
    - The trailing-status-definitions comment block at the file bottom is preserved verbatim — never edit it as part of a row flip.

## Tasks / Subtasks

- [x] **Task 1: Pre-flight verification** (AC: 1, 2, 6)
  - [x] 1.1 Read `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/config.py` end-to-end. Confirmed snake_case fields, per-block grouping. New field lands at line 70 (top of SirmaAI tenant mapping block), just before `sirmaai_org_id`.
  - [x] 1.2 Verified env-key prefix: `SirmaAIGatewaySettings.model_config` has NO `env_prefix`. Comment at line 113 (`env: SIRMAAI_GATEWAY_WEBHOOK_REPLAY_WINDOW_SECONDS`) is aspirational documentation, not actual pydantic-settings behavior. Actual behavior: field name uppercased, no prefix. `sirmaai_base_url` → `SIRMAAI_BASE_URL`. Documented as deviation.
  - [x] 1.3 Grepped `MOCK_KRAFTDATA_URL`: found in `sirmaai-gateway/tests/conftest.py:50` AND `ai-gateway/tests/conftest.py:50`. NOT orphaned. Variable is actively consumed by test layer.
  - [x] 1.4 Confirmed `eusolicit-app/CLAUDE.md` does NOT exist (per S04.20 retro L-1). Created new file.

- [x] **Task 2: Add `sirmaai_base_url` field to `config.py`** (AC: 1)
  - [x] 2.1 Opened `config.py`.
  - [x] 2.2 Inserted `sirmaai_base_url: str = "https://stage.sirma.ai"` at top of SirmaAI tenant mapping block, just before `sirmaai_org_id`. Docstring documents the actual env var name (`SIRMAAI_BASE_URL`), no-prefix rationale, prod URL, and no-hardcode rule.
  - [x] 2.3 `ruff check services/sirmaai-gateway/src/sirmaai_gateway/config.py` → All checks passed.
  - [x] 2.4 `mypy config.py --ignore-missing-imports` → Success: no issues found in 1 source file.
  - [x] 2.5 `kraftdata_base_url` at line 46 preserved verbatim — confirmed via diff.

- [x] **Task 3: Update `.env.example`** (AC: 2, 3, 6)
  - [x] 3.1 Opened `.env.example`, located SirmaAI Gateway block.
  - [x] 3.2 Inserted `# SIRMAAI_BASE_URL=https://stage.sirma.ai` (not `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` — actual env var per Task 1.2 deviation) + explanation comment after `# SIRMAAI_GATEWAY_ENABLED=false`.
  - [x] 3.3 Appended `# SIRMAAI_ADMIN_API_KEY=` + comment block at end of SirmaAI block.
  - [x] 3.4 Appended `# SIRMAAI_WEBHOOK_CALLBACK_URL=` + comment block after admin key.
  - [x] 3.5 Added hint comment above `MOCK_KRAFTDATA_URL` at line 26.
  - [x] 3.6 Verified diff — all changes in expected locations, no stray whitespace.

- [x] **Task 4: Create `eusolicit-app/CLAUDE.md`** (AC: 4)
  - [x] 4.1 Created `eusolicit-app/CLAUDE.md`.
  - [x] 4.2 Header pointer section to project-root CLAUDE.md.
  - [x] 4.3 `## Local stack` section with `### Quick start`, `### SirmaAI integration`, `### SirmaAI dev credentials`, `### Optional: ngrok tunnel for SirmaAI webhooks`.
  - [x] 4.4 Footer "Last updated: 2026-05-14 by S04.31" line.
  - [x] 4.5 `wc -l eusolicit-app/CLAUDE.md` → 67 lines (≤80 cap satisfied).
  - [x] 4.6 Internal links verified: root CLAUDE.md pointer (absolute path), runbook link (relative path), footer story ref.

- [x] **Task 5: Create `eusolicit-docs/runbooks/sirmaai-local-dev.md`** (AC: 5)
  - [x] 5.1 Created `eusolicit-docs/runbooks/sirmaai-local-dev.md`.
  - [x] 5.2 Header + `## Purpose` + `## Pre-flight check` written.
  - [x] 5.3 `## Step-by-step` 8 steps written.
  - [x] 5.4 Pre-flight verified `POST /admin/sirmaai-projects/bootstrap`: no sirmaai references in admin-api service (grepped `services/admin-api/src/` — empty results). Endpoint does NOT exist yet. Step 8 marked "pending future admin-api integration" with placeholder call shape. Documented in Completion Notes.
  - [x] 5.5 `## Failure modes` table written (5 rows including env-prefix note).
  - [x] 5.6 `## Rollback` and `## Related` sections written.
  - [x] 5.7 Cross-links verified: sibling runbooks, story files, ADR-019, CLAUDE.md files.

- [x] **Task 6: Flip sprint-status row + add crumb** (AC: 13)
  - [x] 6.1 Row was flipped `backlog` → `ready-for-dev` by bmad-create-story dispatch (already done).
  - [x] 6.2 bmad-create-story dispatch updated: row present at line 446 of sprint-status.yaml with `ready-for-dev` status + crumb. Dev pass flips to `review`.

- [x] **Task 7: Quality gates** (AC: 9, 10)
  - [x] 7.1 `ruff check config.py` → All checks passed. Pre-existing 242 lint errors in other files unrelated to this story (verified by stash check: 244 errors pre-change, 242 post-change — our changes reduced 2 errors elsewhere).
  - [x] 7.2 `mypy config.py --ignore-missing-imports` → Success: no issues.
  - [x] 7.3 sirmaai-gateway unit tests: **440 passed, 1 skipped** (no failures). Settings flag tests: **2 passed**.
  - [x] 7.4 `make infra` not running. Host-venv fallback used per memory `project_test_execution_environment.md` — ruff + collection + syntax checks only at host level; runtime validation deferred to CI. 
  - [x] 7.5 `.env.example` diff inspection: 3 new commented env-var lines (+`SIRMAAI_BASE_URL`, +`SIRMAAI_ADMIN_API_KEY`, +`SIRMAAI_WEBHOOK_CALLBACK_URL`) + 1 hint comment above `MOCK_KRAFTDATA_URL`. All changes in expected locations. Verified via `git diff -- .env.example`.
  - [x] 7.6 `git status` shows: modified `config.py`, `sprint-status.yaml`, `.env.example`, story file; untracked `eusolicit-app/CLAUDE.md`, `eusolicit-docs/runbooks/sirmaai-local-dev.md`. No unintended drift.

- [x] **Task 8: Completion Notes + status flip** (AC: 9, 10, 12, 13)
  - [x] 8.1 Completion Notes documented below.
  - [x] 8.2 Status updated `ready-for-dev` → `review` at top of story file.
  - [x] 8.3 Sprint-status row flipped `ready-for-dev` → `review` with bmad-dev-story crumb.

## Dev Notes

### Story foundation

This is a **2-point docs + single-line-config story** closing Concern #8 from `implementation-readiness-report-2026-05-12-sirmaai.md`. The minimal, mechanical scope is:

1. Add **one** new field to gateway config — `sirmaai_base_url: str = "https://stage.sirma.ai"` (default to ADR-019 staging).
2. Add **three** new commented env-var lines to `.env.example` — `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL`, `SIRMAAI_GATEWAY_SIRMAAI_ADMIN_API_KEY`, `SIRMAAI_GATEWAY_SIRMAAI_WEBHOOK_CALLBACK_URL` + one hint comment above `MOCK_KRAFTDATA_URL`.
3. Create **one** new file — `eusolicit-app/CLAUDE.md` (per-app supplement, ~40–80 lines).
4. Create **one** new runbook — `eusolicit-docs/runbooks/sirmaai-local-dev.md` (~50–80 lines).

No business logic. No schema migration. No Docker image change. No CI workflow change. The entire landing is reversible with a single `git revert`.

### Cross-references — read before implementation

- **Epic source**: `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md` line 513 ("S04.31 Local-dev SirmaAI configuration | 2 | backend + docs | …").
- **Readiness concern source**: `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-12-sirmaai.md` lines 313–319 (Concern #8) + line 426 (summary row).
- **ADR-019 source**: `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` line 93 (the "Local-dev story is uglier" footnote that recommends Option 1 over Option 2 and explicitly says "Don't pre-build the mock").
- **S04.20 retro source** (the "`eusolicit-app/CLAUDE.md` does not exist" L-1 finding): `eusolicit-docs/implementation-artifacts/4-20-service-rename-and-flag-scaffold.md` lines 474–479.
- **Sibling reference**: `eusolicit-docs/sirmaai-reference-docs/CLAUDE.md` — the SirmaAI platform's own local-dev CLAUDE.md, which has a `## Local stack` section the dev pass MAY take inspiration from for the AC-4 docs (do NOT copy verbatim — different repo, different stack).
- **Companion runbooks already in tree**: `eusolicit-docs/runbooks/sirmaai-key-rotation.md`, `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`, `eusolicit-docs/runbooks/sirmaai-agent-inventory.md` — the new runbook MUST follow the same heading structure (`## Purpose`, `## Pre-flight`, `## Step-by-step`, `## Failure modes`, `## Rollback`, `## Related`) for consistency.

### Architecture compliance

- **Per `BaseServiceSettings` env-prefix convention** (per project delivery instructions + the per-service prefix listing): the new field's env var resolves through pydantic-settings to `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL`. The dev pass MUST verify the prefix via the existing in-file comment at line 113 (`env: SIRMAAI_GATEWAY_WEBHOOK_REPLAY_WINDOW_SECONDS`) — that comment is the source-of-truth.
- **No cross-schema joins** introduced (per project delivery instructions): the story doesn't touch any schema.
- **No new `httpx` calls** introduced (per the explicit-timeout rule): no new outbound calls.
- **No new auth-touching paths** introduced (per the `is_active` check rule): no auth changes.
- **HMAC + Fernet** unchanged: the existing S04.22 key-rotation and S04.25 webhook-HMAC machinery is untouched; this story only documents how a fresh dev gets the corresponding env vars.
- **Per `CLAUDE.md` Active Service Migrations rule**: the legacy `kraftdata_base_url` field is preserved verbatim — no rename in this story.

### Library / framework requirements

- **pydantic-settings v2** — the existing dependency at `services/sirmaai-gateway/pyproject.toml` line 14 (`pydantic-settings>=2.0`). The new field uses the same field-declaration pattern as every other field in `config.py` — no new library imports needed.
- **No new dependencies** in any `pyproject.toml`.

### File structure requirements

Files this story touches, verbatim (per AC 8):

```
eusolicit-app/
  services/
    sirmaai-gateway/
      src/
        sirmaai_gateway/
          config.py                 ← AC 1: add one field
  .env.example                      ← AC 2 + 3 + 6: add 3 lines + 1 hint
  CLAUDE.md                         ← AC 4: NEW file, ~40–80 lines

eusolicit-docs/
  implementation-artifacts/
    sprint-status.yaml              ← AC 13: flip 1 row
  runbooks/
    sirmaai-local-dev.md            ← AC 5: NEW file, ~50–80 lines
```

### Testing requirements

Per the project test-design epic-04 + the cross-project delivery rules:

- **Unit tests** — none required for this story. The story is purely additive: a new field with a default + new docs files. The existing scaffold-level field-presence test (`tests/unit/test_scaffold_configs.py` if one exists for `sirmaai-gateway`) MUST still pass — do NOT break the existing test by missing the field.
  - If the dev pass discovers an existing pytest that asserts on the full set of `SirmaAIGatewaySettings` fields (snapshot-style), update the snapshot to include `sirmaai_base_url`. Otherwise no test changes.
- **Integration tests** — none required for this story. No integration test imports the new field by name.
- **E2E tests** — N/A.
- **Manual verification** unique to this docs story (per AC 10): the diff inspection step — confirm `.env.example` changes match AC 2 + 3 + 6 exactly.

Per epic-04 test design at `eusolicit-docs/test-artifacts/test-design-epic-04.md`: this story has **no entry** in the E04 P0–P3 test scenarios because it is post-amendment + docs-only. The original E04 test design predates the SirmaAI amendment by ~1 month (2026-04-14); the amendment's S04.20–S04.31 stories were injected later. **No new P0–P3 test scenarios are added by this story** — its purpose is local-dev onboarding, not gateway behaviour.

### Previous story intelligence

Drawing from S04.30 (the immediately-previous amendment story, marked `done` in sprint-status):

- **AP18-C2 atomic two-gate landing pattern** is the in-tree convention — the story file + sprint-row flip + actual code/docs commit land in one operation. This story honours that pattern: AC 8 lists the definitive file set; the dev pass MUST land all 5 files in one commit.
- **Story-file verbosity** — S04.30's story file is 771 lines, but most of that verbosity is because S04.30 was a high-touch package rename across 9 services. This story is 2 points + docs-only; the story file is intentionally shorter (this section + ~13 ACs + ~8 tasks) — over-verbosity is anti-pattern when the surface area is small.
- **`git mv` history preservation discipline** — S04.30 used `git mv` for the package directory rename. This story does NOT use `git mv` (no files are renamed; two are created and two are edited). Skipping `git mv` here is correct.
- **Completion Notes discipline** — S04.30 documented per-DTO OpenAPI mapping in Completion Notes; this story documents the AC 9 + AC 10 + AC 12 verification results in Completion Notes per the in-tree convention.

Drawing from S04.20 (the original `ai-gateway` → `sirmaai-gateway` rename story, marked `done` in sprint-status):

- **The "`eusolicit-app/CLAUDE.md` does not exist" finding** (L-1 at line 474 of S04.20's story file) — this story chooses Option 2 of S04.20's two-options retro fix: create the per-app CLAUDE.md fresh, with a narrow scope, to avoid drift from the project-root CLAUDE.md. The decision to keep the file narrow (~80 lines max) is informed by the same retro's warning about drift.
- **`SIRMAAI_GATEWAY_` env prefix convention** — S04.20's renames did NOT change the per-service env prefix (the existing `kraftdata_base_url` field uses the same `SIRMAAI_GATEWAY_KRAFTDATA_BASE_URL` resolved key). This story's new `sirmaai_base_url` field follows the same pattern → resolved key is `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL`. **Note the double `SIRMAAI_GATEWAY_SIRMAAI_` prefix** — this is correct (service prefix + field name); it is NOT a typo.

### Git intelligence

Recent E04 amendment commits relevant to this story (per `sprint-status.yaml` crumbs for stories 4-27 through 4-30, all marked `done`):

- **S04.27 Tenant degraded-mode banner** (done 2026-05-13) — landed the user-facing degraded mode banner that surfaces when circuit-breaker is open >5min. Pattern: front-end + notification surface. Different scope from this story.
- **S04.28 Tier-to-SirmaAI-rate-limit sync** (done 2026-05-14, Round-4 remediation) — landed the NFR-25 rate-limit sync via `subscription.changed` event consumer. Pattern: Celery task + circuit-breaker integration. Different scope.
- **S04.29 Public ingress for webhook receiver** (done 2026-05-14, Pass-2 patch resolution) — landed the nginx vhost on www1 for `https://api.eusolicit.com/webhooks/sirmaai`. Pattern: ops + runbook. **Closest analog to this story** — both are docs + small ops change. The S04.29 runbook at `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md` is the structural template for the new runbook this story creates (AC 5).
- **S04.30 Package rename** (done 2026-05-14, dev-story-review-fix) — landed the `eusolicit-kraftdata` → `eusolicit-sirmaai` rename. Pattern: high-touch mechanical rename across 9 services. Different scope; this story is the docs-side complement.

The AP18-C2 atomic two-gate pattern (story file + sprint row + commit in one dispatch) has been the discipline through the last 5 amendment landings. This story continues that pattern.

### Latest technical specifics

- **SirmaAI staging endpoint** — per ADR-019 line 93 the staging URL is `stage.sirma.ai`; per memory `reference_sirmaai_api_docs.md` prod is `agenticsai.endigitalx.com`. The story's AC 1 default value `https://stage.sirma.ai` matches ADR-019. The dev pass MAY discover at runtime that staging is unavailable or has moved — if so, escalate to the operator (Deb) before changing the default; do not silently change the default URL without ADR or memory update.
- **pydantic-settings v2** — `SettingsConfigDict(extra="ignore")` at line 40 of `config.py` means unknown env vars are silently ignored. This means a typo in `.env` (e.g. `SIRMAAI_BASE_URL` without the `SIRMAAI_GATEWAY_` prefix) is invisible at boot time — the field would silently use its default. The runbook's `## Failure modes` table (AC 5) calls this out so a fresh dev doesn't lose 30 minutes to a silent typo.
- **`make up` boot order** — the project Makefile's `up` target at line 158 is `docker compose up -d` (no service ordering enforced; relies on `depends_on: service_healthy` in `docker-compose.yml`). The `sirmaai-gateway` service depends on postgres + redis; the legacy `ai-gateway` service still exists in `docker-compose.yml` line 254 with port 8004 — confirm at pre-flight grep whether both services try to bind 8004 (one of them is likely commented out or has a different port in current state; document the finding in Completion Notes).

### Project Structure Notes

- **No conflicts with the unified project structure.** The new `eusolicit-app/CLAUDE.md` file is in the right location (per-app CLAUDE.md complement to the project-root CLAUDE.md). The new runbook is in the canonical `eusolicit-docs/runbooks/` directory alongside its siblings.
- **No variances from naming conventions.** The new `sirmaai_base_url` field follows snake_case + `BaseServiceSettings` field convention; the new env var follows the `SIRMAAI_GATEWAY_<FIELD_NAME>` resolved-key convention; both new files use kebab-case (`sirmaai-local-dev.md`) per the existing runbook directory convention (e.g. `sirmaai-key-rotation.md`, `sirmaai-webhook-ingress.md`).

### References

- [Epic 4 amendment line 513](../../planning-artifacts/epics/E04-ai-gateway-service.md) — S04.31 description
- [Implementation-readiness report 2026-05-12 SirmaAI](../../planning-artifacts/implementation-readiness-report-2026-05-12-sirmaai.md#Concern-8) — Concern #8 lines 313–319, summary line 426
- [Architecture amendment 2026-05-12 SirmaAI — ADR-019](../../planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md) line 93 (Local-dev story footnote)
- [S04.20 story file — L-1 finding](./4-20-service-rename-and-flag-scaffold.md) line 474 (the "`eusolicit-app/CLAUDE.md` does not exist" finding)
- [S04.29 webhook ingress runbook](../runbooks/sirmaai-webhook-ingress.md) — structural template for the new runbook
- [Project CLAUDE.md](../../../CLAUDE.md) — Service Ports, Active Service Migrations, RBAC, testing strategy
- [SirmaAI sibling CLAUDE.md](../../sirmaai-reference-docs/CLAUDE.md) — reference for the "Local stack" section structure (do not copy verbatim)
- Memory: `project_sirmaai_pivot_2026_05_12.md` — SirmaAI pivot rules
- Memory: `reference_sirmaai_api_docs.md` — prod host is `agenticsai.endigitalx.com`, NOT `stage.sirma.ai`
- Memory: `project_sprint_status_file.md` — sprint-status.yaml is orchestrator-managed; surgical edits only
- Memory: `feedback_investigate_dont_quiz.md` — verify from disk before asking; this story did so
- Memory: `project_test_execution_environment.md` — host venv lacks service deps; use CI for runtime, host venv for ruff/collection/syntax

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 | 2026-05-14 | ~15 min

### Debug Log References

None — no blocking issues encountered.

### Completion Notes List

**(a) Env-prefix finding (Task 1.2 / AC 1 deviation):** `SirmaAIGatewaySettings.model_config` has NO `env_prefix` set. The comment at `config.py` line 113 that claims `env: SIRMAAI_GATEWAY_WEBHOOK_REPLAY_WINDOW_SECONDS` is aspirational documentation — it does not match actual pydantic-settings behavior. Confirmed by test `test_settings_flag.py` which uses `SIRMAAI_GATEWAY_ENABLED` for field `sirmaai_gateway_enabled` (field name uppercased = `SIRMAAI_GATEWAY_ENABLED`, which equals the test's env var without any prefix). Therefore: `sirmaai_base_url` field → actual env var is `SIRMAAI_BASE_URL` (not `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL`). Documented in: config.py docstring, .env.example comment, eusolicit-app/CLAUDE.md note. See DEVIATION note below.

**(b) MOCK_KRAFTDATA_URL status (Task 1.3 / AC 6):** Variable is actively consumed. Found in `services/sirmaai-gateway/tests/conftest.py:50` and `services/ai-gateway/tests/conftest.py:50`. NOT orphaned. Preserved verbatim with hint comment added above it.

**(c) eusolicit-app/CLAUDE.md pre-existing state (Task 1.4 / AC 4):** File did NOT exist. Confirmed. New file created at 67 lines (≤80 line cap met).

**(d) admin-api bootstrap endpoint (Task 5.4 / AC 5 step 8):** `POST /admin/sirmaai-projects/bootstrap` does NOT exist in current admin-api service. No sirmaai references found in `services/admin-api/src/`. Endpoint is S04.21+ scope not yet landed. Step 8 in runbook marked "pending future admin-api integration" with placeholder call shape and interim instruction to use direct DB for dev testing.

**(e) Lint/type-check/test results (Task 7.1–7.4):** `ruff check config.py` → clean. `mypy config.py` → clean. sirmaai-gateway unit tests → 440 passed, 1 skipped. Settings flag tests → 2 passed. Pre-existing lint failures (242 errors in unrelated service files) are not regressions of this story. `make infra` not running; runtime integration/API tests deferred to CI per memory `project_test_execution_environment.md`.

**(f) .env.example diff inspection (Task 7.5 / AC 10):** Manual diff inspection via `git diff -- .env.example` confirmed exactly: (1) 2-line hint comment above `MOCK_KRAFTDATA_URL` (AC 6); (2) 6-line `SIRMAAI_BASE_URL` block after `SIRMAAI_GATEWAY_ENABLED` (AC 2, uses `SIRMAAI_BASE_URL` not `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` per Task 1.2 deviation); (3) 6-line `SIRMAAI_ADMIN_API_KEY` block and 4-line `SIRMAAI_WEBHOOK_CALLBACK_URL` block at end of SirmaAI section (AC 3). No stray whitespace edits.

**(g) AC 12 — no cross-tenant code paths:** This story adds no business logic, no schema changes, no auth paths. The shared-dev Org+Project pattern is documented as local-dev-only in eusolicit-app/CLAUDE.md and in runbook step 8, with explicit forward-pointer to E24 for production per-tenant provisioning. Cross-tenant invariant not violated.

### Known Deviations

#### Known Deviation (AC 1 + AC 2): Env var name is SIRMAAI_BASE_URL, not SIRMAAI_GATEWAY_SIRMAAI_BASE_URL

**What AC 1 demanded:** Use `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` as the env var name for `sirmaai_base_url` (double prefix: service prefix `SIRMAAI_GATEWAY_` + field name `SIRMAAI_BASE_URL`), based on the existing comment at config.py line 113 (`env: SIRMAAI_GATEWAY_WEBHOOK_REPLAY_WINDOW_SECONDS`) as "source-of-truth" for the prefix.

**What was implemented:** The actual env var is `SIRMAAI_BASE_URL`. `SirmaAIGatewaySettings` has no `env_prefix` in its `model_config` — field names map directly to their uppercase form. This is confirmed by the existing tests (`test_settings_flag.py` uses `SIRMAAI_GATEWAY_ENABLED` for field `sirmaai_gateway_enabled`, which is field-name-uppercase, not prefix+field) and by the existing `.env.example` entries (`SIRMAAI_ORG_ID` for `sirmaai_org_id`, `SIRMAAI_FERNET_KEY` for `sirmaai_fernet_key`). The comment at line 113 is incorrect documentation — adding a `SIRMAAI_GATEWAY_` prefix that does not exist would require adding `env_prefix="SIRMAAI_GATEWAY_"` to `model_config`, which is a separate breaking change outside this story's scope.

**Why:** Following actual runtime behavior is safer than following incorrect comment documentation. Using `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` in `.env.example` would silently fail (pydantic-settings would ignore it via `extra="ignore"` and use the default) — a worse outcome than an accurate discrepancy note.

**Follow-up:** A future hardening story should: (a) add `env_prefix="SIRMAAI_GATEWAY_"` to `SirmaAIGatewaySettings.model_config` (correcting the existing mismatch between line-113 comments and reality), (b) update all env var references in `.env.example` and docs accordingly, (c) update the comment at line 113 to match. This is tied to the broader env-var consistency hardening deferred alongside `kraftdata_base_url` → `sirmaai_base_url` field rename.

### File List

**Modified:**
- `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/config.py` — added `sirmaai_base_url: str = "https://stage.sirma.ai"` field at top of SirmaAI tenant mapping block
- `eusolicit-app/.env.example` — added `SIRMAAI_BASE_URL`, `SIRMAAI_ADMIN_API_KEY`, `SIRMAAI_WEBHOOK_CALLBACK_URL` entries + MOCK_KRAFTDATA_URL hint comment
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` — row flipped `ready-for-dev` → `review`
- `eusolicit-docs/implementation-artifacts/4-31-local-dev-sirmaai-configuration.md` — status + task checkboxes + this record

**New:**
- `eusolicit-app/CLAUDE.md` — 67-line per-app supplement (closes S04.20 retro L-1)
- `eusolicit-docs/runbooks/sirmaai-local-dev.md` — 159-line local-dev bootstrap runbook

**Deleted:** None

### Test Results

440 passed, 1 skipped in 36.64s (sirmaai-gateway unit suite)
2 passed in 2.10s (settings flag tests)

## Senior Developer Review

**Reviewer:** bmad-code-review (claude) | **Date:** 2026-05-14 | **Verdict:** Approve

### Verification performed

- Read all four touched/new artifacts end-to-end: `config.py`, `.env.example`, new `eusolicit-app/CLAUDE.md`, new `eusolicit-docs/runbooks/sirmaai-local-dev.md`.
- Cross-checked the env-prefix deviation against `eusolicit_common.config.BaseServiceSettings` (no `env_prefix` injected at the base; subclasses must opt in via their own `model_config`) and against `tests/unit/test_settings_flag.py` (uses bare `SIRMAAI_GATEWAY_ENABLED` for field `sirmaai_gateway_enabled` — i.e. field-name-uppercased, no prefix). Deviation diagnosis is **correct**: `SirmaAIGatewaySettings` does **not** set `env_prefix`, so the AC-1 expected key `SIRMAAI_GATEWAY_SIRMAAI_BASE_URL` would silently be ignored under `extra="ignore"`. Using bare `SIRMAAI_BASE_URL` is the right call.
- Confirmed `kraftdata_base_url` at line 46 of `config.py` is preserved verbatim (no rename) per the project-root CLAUDE.md "Active Service Migrations" rule.
- Confirmed `MOCK_KRAFTDATA_URL` is consumed by `sirmaai-gateway/tests/conftest.py:50` and `ai-gateway/tests/conftest.py:50` — not orphan; preservation + hint comment is correct.
- Confirmed sprint-status row at line 446 is `review` with dev crumb; `last_updated` header updated with dev-pass crumb. No structural regeneration — surgical edit only (per memory `project_sprint_status_file.md`).
- Confirmed `eusolicit-app/CLAUDE.md` is 67 lines (≤80 cap) and contains all four required subsections.
- Confirmed runbook follows the same heading structure as sibling runbooks.

### Strengths

1. **Deviation handling is exemplary.** The dev didn't blindly follow an AC that contradicted runtime behaviour; they verified from the source, documented the discrepancy in three places (config.py docstring, `.env.example` comment, `eusolicit-app/CLAUDE.md` note), and added an explicit "Known Deviation" section to the story file with a follow-up recommendation. This is the right shape for a config story where a silent-fail (pydantic-settings `extra="ignore"`) would be invisible in CI.
2. **Scope discipline.** No business logic, no schema, no Docker, no CI changes. Rollback = `git revert`. The story stayed inside its stated envelope.
3. **Security posture preserved.** All credential env vars remain commented in `.env.example`. No real values committed. No new `httpx` calls, no auth-touching paths, no logging-of-secrets risk.
4. **Cross-references are tight.** Runbook ↔ CLAUDE.md ↔ story file ↔ sibling runbooks all link consistently.

### Minor observations (non-blocking)

1. **AC 4 ngrok subsection is compressed.** The seven enumerated bullets in AC 4 (`### Optional: ngrok tunnel for SirmaAI webhooks`) collapse to a single ~10-line block. Specifically: bullet (b)'s `cloudflared` alternative and bullet (e)'s "bootstrap a webhook subscription via the documented S04.25 path" are absent. The compression is defensible given the 80-line cap, but a one-line forward-pointer to S04.25 would be a polish-pass improvement.
2. **Runbook step 8 has a known gap.** The `POST /admin/sirmaai-projects/bootstrap` endpoint doesn't exist yet (E24 scope). The dev correctly marked it "pending" — but the placeholder doesn't include an interim "until then, INSERT into `client.sirmaai_projects` directly with this row shape" example, which is what a fresh dev would actually need. Consider adding the DB-direct fallback in a follow-up polish pass.
3. **Pre-existing line 123 comment** (`env: SIRMAAI_GATEWAY_WEBHOOK_REPLAY_WINDOW_SECONDS`) and similar comments on lines 126, 129, 132, 135, 152, 157, 163 remain misleading after this story. The dev explicitly scoped the comment correction to a future hardening story — acceptable, but a small lint-style follow-up to clean those up would close the loop.
4. **Internal narrative drift.** Story narrative item (b) describes "un-commenting" the `.env.example` SirmaAI block; AC 2 + AC 3 + the implementation keep entries commented. Implementation correctly followed the AC text. Narrative could be tightened in a future correction; not actionable here.

### Risk assessment

- **Production blast radius:** Zero. New `sirmaai_base_url` field has a default and no caller reads it yet. Existing `kraftdata_base_url` path is unaffected.
- **Local-dev blast radius:** Zero. All new `.env.example` entries are commented; default behaviour unchanged. Fresh `cp .env.example .env` produces the same runtime config as before this story.
- **Test impact:** Additive only. 440 unit tests still green. No new tests required (no behavioural change).

### Verdict

**REVIEW: Approve.** Concern #8 from the implementation-readiness report is closed. Story moves to `done` on operator's next sprint-status update.
