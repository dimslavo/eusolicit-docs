# Story 7.17: Proposal Generate Endpoint — Security Error Sanitization & Audit Trail Hardening

Status: draft

**Origin:** NFR Assessment Epic 7 (`eusolicit-docs/test-artifacts/nfr-report.md`, 2026-04-18) — **Overall FAIL**, HALT recommended
**Priority:** **P0 / BLOCKER** — epic retrospective cannot close with NFR4 FAIL + NFR12 FAIL outstanding
**Epic:** E07 hot-fix / hardening backlog (pre-demo milestone sign-off)
**Points:** 3 | **Type:** backend
**NFR Category:** **Security (NFR4)** — primary; bundles **Compliance / Audit Trail (NFR12)** on the same endpoint
**Affected Service:** `client-api` (endpoint `POST /api/v1/proposals/:id/generate`); indirect touch on `ai-gateway` error contract. No changes to `admin-api`, `data-pipeline`, `notification`, or `enterprise-api`.

---

## Story

As a **platform security & compliance owner**,
I want the **`POST /proposals/:id/generate` SSE endpoint to emit sanitized error events AND write an immutable audit-trail record for every generation mutation**,
so that **internal AI-Gateway stack traces, upstream credentials, and infrastructure details are never leaked to authenticated tenants, and every proposal version produced by AI generation is traceable in `shared.audit_log` as required by project-context.md Rule 44**.

---

## NFR-Specific Measurable Targets

| Target | Category | Measurement | Acceptance Threshold |
|---|---|---|---|
| **Error payload sanitization** | Security (NFR4) | Shape of every SSE `event: error` frame emitted by `POST /generate` | 100% of error frames match the fixed schema `{"error": "ai_gateway_error", "correlation_id": "<uuid>"}` — zero leakage of stack traces, gateway URLs, API keys, or upstream HTTP bodies. Verified by contract test over ≥ 6 upstream failure modes (5xx, 4xx, timeout, circuit-open, connection-refused, malformed JSON). |
| **Timing-safe error path** | Security (NFR4) | Wall-clock difference between successful-path error and "proposal not found" error | < 5 ms p95 (prevents enumeration via timing oracle) |
| **Audit write coverage** | Compliance (NFR12) | % of `POST /generate` invocations (success + failure) producing a `shared.audit_log` row | 100% — one row per invocation with `entity_type='proposal'`, `action_type='generate'`, `before=<version_n content hash>`, `after=<version_n+1 content hash or null>`, `user_id`, `ip_address`, `correlation_id` |
| **Audit write non-blocking latency** | Performance guardrail for the fix | Added p95 latency on `POST /generate` TTFB vs pre-fix baseline | < +10 ms TTFB (audit write must be fire-and-forget via `asyncio.create_task`, per Rule 45) |
| **Audit write error budget** | Reliability | `audit_log` insert failures tolerated without impacting the user | 100% of DB exceptions caught, logged as `structlog.error("audit_write_failed", …)`, and **not** propagated to the client (never returns 500) |
| **Prompt injection surface reduction** | Security (NFR4, residual E07-R-005) | Content-block body sanitation before forwarding to AI Gateway | Strip/escape `<script>`, null bytes, control chars < 0x20 (except `\n`, `\t`, `\r`); reject bodies > 100 KiB with 422 |
| **SSE response headers** | Security / correctness | `Cache-Control: no-cache`, `X-Accel-Buffering: no`, `X-Content-Type-Options: nosniff` present on error frames too | Verified by integration test on each error branch |

---

## Acceptance Criteria (Given / When / Then)

### AC1 — SSE error event is sanitized (NFR4 primary)

**Given** a client with a valid JWT has triggered `POST /api/v1/proposals/{id}/generate`
**And** the upstream AI Gateway returns `502 {"detail": "Traceback (most recent call last)… API_KEY=sk-xyz …"}`
**When** the client-api proxy consumes the upstream response
**Then** the SSE frame written to the client MUST be exactly `event: error\ndata: {"error":"ai_gateway_error","correlation_id":"<uuid>"}\n\n`
**And** the raw upstream body MUST NOT appear in any SSE frame, HTTP response header, or any log line at level `INFO` or below (it MAY appear once in a structured ERROR log for ops).

### AC2 — Error branch parity across failure modes (NFR4)

**Given** the generation task encounters any of: `httpx.ConnectError`, `httpx.TimeoutException`, `CircuitOpenError`, upstream 4xx, upstream 5xx, malformed SSE chunk, `asyncio.CancelledError` on shutdown
**When** the error bubbles to the SSE dispatcher
**Then** the emitted error frame is byte-identical (except for `correlation_id`) to AC1's schema
**And** `proposals.generation_status` is transitioned to `failed`
**And** the error is mapped through a single `_sanitize_generation_error(exc) -> str` helper (single point of escape).

### AC3 — Audit row is emitted on every mutation (NFR12 / Rule 44)

**Given** a `POST /generate` request completes (success, failure, or client-disconnect)
**When** the request handler (or its `finally` block / `asyncio.create_task`) runs
**Then** exactly **one** row is inserted into `shared.audit_log` with:
- `entity_type = 'proposal'`
- `entity_id = <proposal_id>`
- `action_type = 'generate'`
- `user_id = <current_user.user_id>`
- `company_id = <current_user.company_id>`
- `ip_address = <request.client.host>` (read from `X-Forwarded-For` if configured, see Rule 46 deferral)
- `correlation_id = <x-request-id>`
- `before = {"version_number": N, "content_hash": "<sha256>"}` (or `null` for first generation)
- `after = {"version_number": N+1, "content_hash": "<sha256>"}` for success, or `{"status": "failed", "error_code": "ai_gateway_error"}` for failure
- `created_at = now()`

### AC4 — Audit write never breaks the user flow (Rule 45)

**Given** `shared.audit_log` is unreachable (simulate via broken DB fixture or patched write)
**When** `POST /generate` runs end-to-end
**Then** the client STILL receives a successful SSE stream (or the normal sanitized error frame)
**And** no 500 is ever returned
**And** a single structured `ERROR` log with key `audit_write_failed` is emitted with enough context (`proposal_id`, `correlation_id`, exception type) to allow manual recovery.

### AC5 — Timing-safe 404 on cross-tenant generate (NFR4, ties E07-R-001)

**Given** Company A credentials call `POST /proposals/{B_proposal_id}/generate`
**When** the handler validates ownership
**Then** the wall-clock difference between the cross-tenant 404 and an own-company success-path error is < 5 ms p95 over 100 runs
**And** both return identical error bodies `{"error": "not_found", …}`.

### AC6 — Content-block body is sanitized before AI Gateway forwarding (E07-R-005 residual)

**Given** `build_generation_payload()` assembles content from `content_blocks`
**When** a block's `body` contains null bytes, control chars < 0x20 (except `\n \t \r`), or exceeds 100 KiB
**Then** null bytes / disallowed control chars are stripped
**And** bodies > 100 KiB cause `POST /generate` to return `422 {"error":"content_block_too_large","block_id":"…"}`
**And** sanitization is applied in a dedicated `sanitize_content_block_body()` helper unit-tested with 12 fuzz vectors.

### AC7 — SSE response headers hardened on the error path

**Given** any error frame is emitted
**When** the response headers are inspected
**Then** the following are present: `Cache-Control: no-cache`, `X-Accel-Buffering: no`, `X-Content-Type-Options: nosniff`, `Content-Type: text/event-stream; charset=utf-8` (per Rule 51).

### AC8 — No regression in Epic 7 test suite

**Given** the full Epic 7 test suite (`pytest services/client-api/tests -m "unit or integration or api" -v`)
**When** run in CI after the fix
**Then** all tests pass
**And** coverage on `services/client-api/src/client_api/api/v1/proposals.py` stays ≥ 80%.

---

## Affected Services (impact map)

| Service | Impact | Change |
|---|---|---|
| `client-api` | Primary | Error sanitizer helper, audit-log write, content-block sanitation, header hardening, tests |
| `ai-gateway` | Contract consumer only | No code change — this story **relies on** existing E04 error envelope; documents the assumption |
| `admin-api` | None | — |
| `data-pipeline` | None | — |
| `notification` | None | — |
| `enterprise-api` | None | — |

---

## Test Strategy (aligned with `eusolicit-app/tests/README.md` markers)

| Level | Marker | Path | What it covers |
|---|---|---|---|
| Unit | `@pytest.mark.unit` | `services/client-api/tests/unit/test_sanitize_generation_error.py` | Every failure-mode input → fixed schema output; `_sanitize_generation_error()` is pure. |
| Unit | `@pytest.mark.unit` | `services/client-api/tests/unit/test_sanitize_content_block_body.py` | 12 fuzz vectors (null bytes, control chars, UTF-8 edge cases, 100 KiB boundary). |
| Integration | `@pytest.mark.integration` | `services/client-api/tests/integration/test_generate_audit_log.py` | testcontainers Postgres + respx-mocked AI Gateway; asserts `shared.audit_log` row shape for success, failure, cancellation. Uses transaction-rollback fixture per Epic 2 gold standard. |
| Integration | `@pytest.mark.integration` | `services/client-api/tests/integration/test_generate_error_sanitization.py` | respx `side_effect` cycles all 7 failure modes; parses SSE frames byte-for-byte. |
| Integration | `@pytest.mark.integration` | `services/client-api/tests/integration/test_generate_audit_failopen.py` | Patches audit writer to raise; verifies stream still completes and no 500. |
| API | `@pytest.mark.api` | `tests/api/test_generate_security.py` | Live service; cross-tenant 404 timing assertion (AC5) using `time.monotonic()` over 100 runs. |
| Cross-service | `@pytest.mark.cross_service` | `tests/cross_service/test_generate_sse_headers.py` | Full stack: client-api + ai-gateway + redis. Verifies SSE headers and end-to-end correlation_id propagation. |
| E2E | Playwright | `e2e/specs/proposals/generate-error-display.spec.ts` | UI shows generic "AI Gateway error" toast — never raw stack trace text. Reuses SSE mock from S07.13. |

Coverage gate: 6 new unit tests, 4 new integration tests, 1 new API test, 1 new cross-service test, 1 new Playwright spec. ~13 new tests total.

---

## Definition of Done

- [ ] All 8 acceptance criteria pass in CI
- [ ] `pytest services/client-api/tests -v` green; overall coverage ≥ 80%
- [ ] `ruff`, `mypy`, `make quality-check` clean
- [ ] NFR re-assessment on `nfr-report.md` flips NFR4 from FAIL → PASS and NFR12 from FAIL → PASS; overall Epic 7 NFR status moves from FAIL → PASS (or CONCERNS if load tests still pending — see Deferred section)
- [ ] `shared.audit_log` row presence verified manually in staging with one real `POST /generate` call (screenshot attached to story)
- [ ] `_sanitize_generation_error()` source reviewed by a second engineer — no raw exception fields reach any user-facing surface
- [ ] `project-context.md` "Known Technical Debt" row for *AI Gateway error exposure* removed; audit-trail coverage for proposals added to "Audit Trail" compliance section
- [ ] `deferred-work.md` item "Error SSE event forwards raw AI Gateway data" marked resolved, with link to this story's merged PR
- [ ] Traceability entry added in `traceability-matrix.md` mapping AC1–AC8 to NFR4 / NFR12 and to risks E07-R-003 (residual) and E07-R-005
- [ ] E07 retrospective amended to reference this hot-fix and the cross-epic deferral rule ("Cap deferral at 2 consecutive epics" from Epic 4 anti-patterns)

---

## Epic Placement & Priority Recommendation

- **Target Epic:** **E07 hot-fix backlog (Sprint 8, before epic close)**. NFR assessment recommended **HALT** — the epic cannot be marked `done` in `sprint-status.yaml` while NFR4 / NFR12 are FAIL. Do **not** push this to E08 (billing) or defer to a cleanup sprint: project-context.md anti-pattern "Don't defer NFR HIGH-priority items from one epic to the next without re-surfacing them" explicitly forbids it, and the Epic 4 retro added a hard cap of 2 consecutive deferrals (this would be the 1st, but the underlying item was already deferred once from the S07.05 code review on 2026-04-17 — so accepting another deferral puts us at the cap).
- **Priority:** **P0 / BLOCKER** — marked as HALT by the NFR assessor. Treat as a release-blocking defect for the Demo milestone.
- **Points:** 3 (0.5 day for sanitizer + tests, 1 day for audit write + fail-open wiring, 0.5 day for content-block sanitation, 0.5 day for tests, 0.5 day buffer). Could be split into two 2-point stories (Security hardening vs. Audit trail) if the team prefers, but they touch the same file and share most of the test fixtures, so bundling is cheaper.

---

## Dependencies & Deferred-Item Bundling

### Bundled into this story (resolved together)

From the ~126 tracked deferred items in `deferred-work.md` and `project-context.md` → Known Technical Debt:

1. **[deferred-work.md, S07.05 review, 2026-04-17]** "Error SSE event forwards raw AI Gateway data instead of fixed message (AC6)" — resolved by AC1/AC2.
2. **[project-context.md, Known Tech Debt, Audit Trail row]** "All mutations must write to `shared.audit_log`" — resolved for the `POST /generate` path by AC3/AC4 (keeps the row honest for other endpoints that already comply).
3. **[deferred-work.md, S07.05 review]** "`CancelledError` not caught in `_run_generation_task`" — partially resolved: AC2 mandates the cancel branch route through the sanitizer and flip `generation_status → failed` (removes the 300 s stuck-proposal window for the shutdown case, complementing DW-02's scheduler fix).
4. **[E07-R-005 residual from test-design-epic-07.md]** "Prompt injection / content-block sanitation" — resolved by AC6.

### Explicit hard dependencies

- **DW-02 (Celery Task Infrastructure Fixes)** — independent but related. DW-02 ensures `reset_stuck_proposals_task` actually runs; this story ensures the cancel/failure paths flip `generation_status → failed` so DW-02's job is bounded. Order is not strict, but landing DW-04 first avoids the case where audit rows are missing for proposals DW-02 later resets.
- **No schema migration needed** — `shared.audit_log` already exists (Rule 44, used since E02).

### Explicitly deferred (NOT bundled — call them out)

Keep these in `deferred-work.md` with explicit rationale; do **not** expand this story's scope:

1. **[project-context.md, Known Tech Debt, AI Gateway row]** "Circuit breaker per-instance state (E04-R-005)" — a Redis-backed CB is a separate infrastructure story; not needed to pass NFR4/NFR12 since the sanitizer normalizes `CircuitOpenError` the same as any other failure.
2. **[project-context.md, Audit Trail Rule 46]** "`ip_address` from `request.client.host` is proxy IP" — real `X-Forwarded-For` parsing with proxy allowlist is a cross-cutting infra story (affects every audit-writing endpoint). Record the current-proxy-IP value and file/attach the XFF backlog item.
3. **[project-context.md, Observability row]** "No `x-request-id` correlation header" — this story **uses** `x-request-id` if present but does not add the middleware that guarantees it on every request. Add a fallback: generate a `uuid4()` if the header is missing, and note the middleware debt in the story's Dev Notes.
4. **[nfr-report.md, NFR2/NFR5 CONCERNS]** "No load tests have verified concurrent generation limits yet" — performance NFR is **CONCERNS**, not FAIL. Leave for a dedicated load-test story (precedent: S12.17 template). NFR re-assessment will move NFR4/NFR12 to PASS but NFR2/NFR5 stay at CONCERNS pending that work.
5. **[deferred-work.md, S07.05 review]** "`_version_write_locks` dict grows unboundedly", "`set_generation_status` lacks `Literal`", "`asyncio.run()` in cleanup" — code-quality deferrals, not security/compliance. Leave in deferred-work.md.
6. **[project-context.md, Observability row]** "No Prometheus /metrics endpoint" — cannot be closed here; the audit-write metric (success/fail counter) should be a `structlog` event for now with a TODO to expose it once the `/metrics` scaffold lands.

---

## Dev Notes

### Where the sanitizer should live
`services/client-api/src/client_api/services/proposal_service.py` already owns `build_generation_payload()` and `_run_generation_task()`. Add `_sanitize_generation_error(exc: BaseException) -> tuple[str, str]` returning `(error_code, log_detail)` in the same module. The SSE dispatcher sends only `error_code`; the `log_detail` string feeds a single `structlog.error("generation_upstream_failure", detail=…)` call. This keeps the policy in one place.

### Audit write pattern (reuse E04 retrospective standard)
```python
async def _write_generation_audit(…): ...   # async writer using AsyncSession

# In the request handler, fire-and-forget per Rule 45
asyncio.create_task(_write_generation_audit(…))
```
Catch `Exception` in the writer's top-level `try/except`; NEVER let it bubble. Follow the pattern already in `ai-gateway` webhook logging (E04 retrospective).

### SSE error-frame writer
Extract frame emission into `_emit_sse_error(correlation_id)` that always writes the fixed schema; the proxy loop funnels every exception through it. Makes AC2 and AC7 trivially testable.

### Do NOT skip the timing assertion (Epic 4 anti-pattern)
The Epic 4 retrospective flagged "Don't implement SSE proxy without a wall-clock latency benchmark" — applies here too. AC5's < 5 ms p95 timing check goes in the `tests/api/` live test with `time.monotonic()`.

---

## References

- `eusolicit-docs/test-artifacts/nfr-report.md` — Epic 7 NFR Assessment (FAIL, this story's trigger)
- `eusolicit-docs/project-context.md` — Rules 44, 45, 46 (audit trail); Rules 50, 51 (SSE hardening); "Known Technical Debt" table
- `eusolicit-docs/implementation-artifacts/deferred-work.md` — S07.05 review deferrals
- `eusolicit-docs/test-artifacts/test-design-epic-07.md` — risks E07-R-003, E07-R-005
- `eusolicit-docs/implementation-artifacts/7-5-ai-draft-generation-backend-sse-integration.md` — source spec whose AC6 was violated
- `eusolicit-docs/implementation-artifacts/dw-02-celery-task-infrastructure-fixes.md` — related (cancel path interaction)
- `eusolicit-app/services/client-api/src/client_api/services/proposal_service.py` — target file
- `eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py` — endpoint file
- `eusolicit-app/tests/README.md` — pytest marker conventions
