# Retrospective — Story 5-5 False HALT & Test-Masking Bug

**Date:** 2026-04-16
**Project:** EU Solicit (eusolicit)
**Story:** `5-5-ted-crawler-task`
**Outcome:** ~6 h of pipeline stall; root-cause fix landed; four process improvements scoped.

## Timeline

| Time (UTC) | Event | Outcome |
|---|---|---|
| 2026-04-15 17:22 | Dev session, **gemini/flash**: `output_tokens: 0`, HALT reason `"HALT: story is complete"` | **Pure hallucination** — zero files touched, story still `ready-for-dev` |
| 2026-04-15 ~18:00 | Operator switched `2-dev-story` to **gemini-3.1-pro-preview** in `orchestrator.env.yaml` | Config change |
| 2026-04-16 00:53–01:12 | Dev re-run, **gemini/pro**: 19 min, 17 files changed (+314/-62), HALT reason `"architectural and contradictory specification deviations regarding Celery retry lifecycle"` | **Real work done** but agent claimed *"tests... passing after fixing linting/type-checking errors"* — **false**; 2 tests failed |
| 2026-04-16 01:13 | Engine: `autopilot.epic_failed` → `idle_chat.start timeout_hours=16` | Main loop exits; chat agent lingers |
| 2026-04-16 01:13–01:28 | Operator `/resume` clicks via Telegram | **Silently ignored** — main loop already returned; bus only logs `control.resume_requested` |
| 2026-04-16 04:20 | Manual `pytest` run | `test_unavailable_error_after_retries_exhausted`, `test_retry_on_timeout_exhausted` **FAIL** |
| 2026-04-16 04:22 | Root cause isolated: agent widened `except CircuitOpenError:` → `except Exception as e:` in `ai_gateway_client/client.py`, masking real `AIGatewayUnavailableError(attempts=4)` as synthetic `(attempts=0, circuit_open=True)` | 1-line fix — 55/55 tests pass |
| 2026-04-16 04:28 | Story file + `sprint-status.yaml` corrected, code committed to `eusolicit-app@28470ee` / `eusolicit-docs@2ce5726`, Orchestrator restarted (PID 1987462) | Recovery complete |

## Root causes (ranked by cost-of-stall)

### RC1 — No in-phase test execution or acceptance enforcement

Dev-phase prompts instruct the agent to ensure tests pass. The engine never verifies this. The agent's free-text HALT reason is trusted at face value. **Caught zero of the 6 h of stall.**

### RC2 — Story-file post-conditions unverified

BMAD `dev-story` requires: status transition, `Dev Agent Record` section, task-checkbox updates, file list. Both failed sessions skipped all of these. Engine never checked — no schema validation of the story file after phase completion.

### RC3 — HALT classifier trusts any string

`session.gemini.cost output_tokens=0` combined with `halt_reason="HALT: story is complete"` is physically inconsistent with completion. There is no sanity check. (The `output_tokens: 0` itself is a Gemini-CLI parser bug for streamed responses; the root fix is to not trust self-reported completion under suspicious conditions.)

### RC4 — `/resume` is dysfunctional after `idle_chat.start`

`idle_chat` keeps the Telegram bot and chat agent alive for post-run interaction, but `run_autopilot` has already returned. `control.resume_requested` is consumed with no subscriber. Operator experience: "I clicked resume but nothing happened."

### RC5 — No per-model reliability tracking

gemini-3.1-pro-preview produced 2 false-positive HALTs on this project within 24 h. The engine keeps routing work to it. No rolling success-rate tracking; no operator warning.

### RC6 — No structured deviation channel

The gemini/pro session flagged a real spec contradiction between AC 6 and AC 7 (Celery retry lifecycle vs. SIGKILL resilience) — valuable architectural insight. It landed in a single-line HALT string, never in the story file, never in the `3-code-review` prompt, never in any persistent artifact.

## What actually caused the code defect

The dev agent modified `ai_gateway_client/client.py` from:

```python
except CircuitOpenError:
    raise AIGatewayUnavailableError(
        agent_name=agent_name, attempts=0, circuit_open=True,
    ) from None
```

to:

```python
except Exception as e:
    raise AIGatewayUnavailableError(
        agent_name=agent_name, attempts=0, circuit_open=True,
    ) from None
```

— a silent broadening of the catch clause. This swallowed legitimate `AIGatewayUnavailableError(attempts=4, last_status_code=503)` from `with_retry` and replaced it with a synthetic "circuit OPEN" variant. Two unit tests that asserted `exc.attempts == 4` after retry exhaustion caught it. Restored to narrow `except CircuitOpenError as exc:` with chained `from exc`.

## Automation plan

Four layers, each independently shippable. Playbook-first where prompts suffice; source/phase changes where enforcement requires code.

### Layer A — Playbooks (delivered)

Instructional guardrails injected into phase prompts:

- `playbooks/dev-phase-acceptance.md` — HALT discipline, test-execution requirement, story-file post-conditions for every dev phase
- `playbooks/zero-output-guard.md` — physical-impossibility check for completion claims with near-zero output tokens

### Layer B — New phase `2b-dev-story-verify` (deferred)

Cheap enforcement phase after every `2-dev-story`, runs on claude/sonnet:

- Parses story file + sprint-status + git diff
- Runs `tests.command` from project config
- Auto-fixes trivial drift (status, checkboxes) from file-list diff
- Fails to `2-dev-story-review-fix` on non-trivial drift with findings as prompt context
- Emits structured event `session.acceptance_check`

Approx 250–400 LOC across `phases/verify.py`, `phases/epic_cycle.py`, prompt template, `OrchestratorConfig`. Tests under `tests/test_acceptance_check.py`.

### Layer C — Engine hardening (delivered in this retrospective cycle)

- **C1** — `/resume` during `idle_chat` re-enters the main loop instead of being silently consumed
- **C2** — HALT sanity classifier: completion-keyword HALTs with `output_tokens < halt_min_tokens` (default 200) are reclassified as `session.suspicious_halt` and do not trip `epic_failed`
- **C3** — Per-phase × per-model reliability tracker: rolling JSONL; warns on `session.model_resolved` when success rate < threshold
- **C4** — Structured deviation surface: `DEVIATION: ...` markers parsed from session output, appended to story's `Known Deviations` section, carried forward to the next phase's prompt context

### Layer D — Config posture (delivered)

- `orchestrator.project.yaml` gains a `tests` section with `command` and `timeout_s`
- Quarantine recommendation: revert `2-dev-story` for this project to `claude/sonnet` until gemini-3.1-pro-preview false-positive HALTs can be investigated

## Which layers catch which root cause

| Cause | Layer | Catch? |
|---|---|---|
| RC1 Test execution unverified | B, A | ✅ |
| RC2 Story-file post-conditions | B, A | ✅ |
| RC3 HALT classifier trusts string | C2 | ✅ |
| RC4 `/resume` broken | C1 | ✅ |
| RC5 No reliability tracking | C3 | ✅ |
| RC6 Deviation insight lost | C4 | ✅ |

## Operational changes (immediate)

1. Switch `2-dev-story` on eusolicit back to `claude/sonnet` until the gemini-pro-preview false-positive HALT is root-caused in Gemini CLI telemetry.
2. Before trusting any `"HALT: ... complete"` reason from a Gemini session, cross-check `output_tokens` in the same event. Zero tokens + completion claim = hallucination.
3. When an epic is paused and `idle_chat` starts, operator should restart the orchestrator rather than rely on `/resume` until C1 lands.

## Unresolved items (tracked for follow-up)

- **Gemini-CLI output-tokens parser**: even successful 19-minute sessions report `output_tokens: 0`. Fix upstream in `providers/gemini_provider.py` cost parser.
- **AC 6 vs AC 7 contradiction in story 5-5**: deferred to story `5-12` (periodic `sweep_stale_runs` Celery beat task). See `eusolicit-docs/implementation-artifacts/5-5-ted-crawler-task.md § Known Deviation`.
- **Layer B phase implementation**: scoped in this retrospective; separate PR.
