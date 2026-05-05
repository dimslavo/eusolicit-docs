

## Patterns

- Discovered in Epic 12: ` | — | — | Backend API coverage deep and consistent across all 12 backend stories |

## Patterns

- Discovered in Epic 12: ` | — | — | Tier gate E2E with all 4 tiers is the correct pattern for Professional+/Enterprise features |

## Patterns

- Discovered in Epic 12: ` | — | — | Single `test_ip_allowlist.py` middleware suite applied to all VPN-restricted routes |

## Patterns

- Discovered in Epic 12: ` | — | — | S12.01 is model infrastructure story: 258 tests, all ACs, CLI, migration rollback |

## Anti-Patterns

- Discovered in Epic 12: ` | — | — | Frontend E2E completely absent for 7/18 stories — `bmad-testarch-atdd` skipped for all frontend |

## Anti-Patterns

- Discovered in Epic 12: ` | — | — | Non-functional stories marked done without execution evidence |

## Anti-Patterns

- Discovered in Epic 12: ` | — | — | TEA skipped entire epic — 4th occurrence of TEA under-execution |

## Patterns

- Discovered in Epic 4: ` All 3 high-priority risks mitigated (webhook HMAC, SSE reliability, Redis publish gap)

## Patterns

- Discovered in Epic 4: ` Two-layer resilience composition: `circuit_breaker(retry(http_factory))` — now the standard for all outbound HTTP

## Patterns

- Discovered in Epic 4: ` YAML-driven agent registry decouples logical names from external UUIDs

## Patterns

- Discovered in Epic 4: ` testcontainers + respx is the mandatory backend integration test stack

## Patterns

- Discovered in Epic 4: ` TRACE_GATE PASS on first run (no Run 2 needed)

## Patterns

- Discovered in Epic 4: ` Fire-and-forget async DB writes via `asyncio.create_task()`

## Patterns

- Discovered in Epic 4: ` Structured admin introspection endpoints for all stateful services

## Anti-Patterns

- Discovered in Epic 4: ` **CRITICAL** — Zero ATDD checklists for any Epic 4 stories (0/10)

## Anti-Patterns

- Discovered in Epic 4: ` **HIGH** — No NFR assessment performed for Epic 4

## Anti-Patterns

- Discovered in Epic 4: ` **CRITICAL** — 10 security/reliability hardening items, 4 epics overdue, no action taken

## Anti-Patterns

- Discovered in Epic 4: ` **LOW** — `epic-4: in-progress` in sprint-status despite all stories done (4th occurrence)

## Anti-Patterns

- Discovered in Epic 4: ` **MEDIUM** — S03.11 TEA still `in-progress` (2nd consecutive epic carry-forward)

## Anti-Patterns

- Discovered in Epic 4: ` **MEDIUM** — SSE latency `< 100ms` has no wall-clock timing test

## Anti-Patterns

- Discovered in Epic 4: ` **LOW** — Config negative-path tests for missing env vars not implemented

## Patterns

- Discovered in Epic 5: ` All 3 top-ranked risks (Score ≥6) mitigated with dedicated P0 test coverage — E05-R-001 (dedup race, FULL), E05-R-002 (cascade, PARTIAL), E05-R-003 (Redis publish, SUBSTANTIAL)

## Patterns

- Discovered in Epic 5: ` TRACE_GATE PASS with 100% P0/P1 FULL coverage (39/39) on first run — consistent with E04 result

## Patterns

- Discovered in Epic 5: ` Structured deferred-work documentation with file location, line refs, and rationale enables zero-archaeology technical debt management

## Patterns

- Discovered in Epic 5: ` NFR assessment completed at epic close (closed E04 carry-forward) — test design → traceability → ATDD → code review → NFR → retrospective is the correct gate sequence

## Patterns

- Discovered in Epic 5: ` Celery `task_always_eager` + testcontainers PostgreSQL + Redis + respx + factory fixtures is the proven template for all backend async/worker service integration tests

## Patterns

- Discovered in Epic 5: ` E04 two-layer resilience pattern (circuit_breaker outer, retry inner) reused successfully in E05 pipeline AI Gateway client without redesign

## Anti-Patterns

- Discovered in Epic 5: ` **CRITICAL** — `_RUN_ID_REGISTRY` module-level dict is Celery prefork-incompatible; `on_failure` handler cannot retrieve run_id across process boundary; all tests run in eager mode masking this; breaks CrawlerRun lifecycle management in production

## Anti-Patterns

- Discovered in Epic 5: ` **HIGH** — Non-atomic DB transactions in all 3 crawlers and enrichment queue worker; crash between commits leaves data committed but status permanently `running` or `processing`; must be story AC not code review finding

## Anti-Patterns

- Discovered in Epic 5: ` **HIGH** — S05.01–S05.05 ATDD checklists not created (5/12 stories); infrastructure/schema stories are not exempt from ATDD before-dev gate

## Anti-Patterns

- Discovered in Epic 5: ` **MEDIUM** — No TEA test review scores for any E05 story; `tea_status` in sprint-status.yaml has no E05 entries (gap carried from E04)

## Anti-Patterns

- Discovered in Epic 5: ` **MEDIUM** — OBS-001: 4xx errors incorrectly increment circuit breaker failure counter in both E04 AI Gateway service and E05 pipeline client; same logical bug in two places; fix must be applied simultaneously to both

## Anti-Patterns

- Discovered in Epic 5: ` **MEDIUM** — Silent `except Exception: pass` on all Prometheus metrics instrumentation across 4 pipeline files; hides operational errors without any log signal

## Anti-Patterns

- Discovered in Epic 5: ` **MEDIUM** — No k6 pipeline throughput baseline for third consecutive epic; `load-test-results.md` remains unfilled template; 24h freshness SLA unverifiable; must become sprint deliverable not backlog item

## Anti-Patterns

- Discovered in Epic 5: ` **LOW** — `pipeline_health` 503 response exposes `str(exc)` including connection strings and hostnames; defence-in-depth violation even on ClusterIP-only endpoint

## Patterns

- Discovered in Epic 5: ` entries and 7 new `[ANTI-PATTERN]` entries discovered in Epic 5

## Patterns

- Discovered in Epic 5: ` | — | Risk mitigation discipline, TRACE_GATE consistency, structured debt tracking, NFR gate restored, Celery eager test stack, E04 resilience pattern reuse |

## Anti-Patterns

- Discovered in Epic 5: ` entries discovered in Epic 5

## Anti-Patterns

- Discovered in Epic 5: ` → `[ACTION]` | **critical** | `_RUN_ID_REGISTRY` module-level dict breaks Celery prefork — masked by eager-mode tests, silent in production · `IMPACT: standards_update` |

## Anti-Patterns

- Discovered in Epic 5: ` → `[ACTION]` | **high** | Non-atomic DB transactions in all 3 crawlers + enrichment worker · `IMPACT: standards_update` |

## Anti-Patterns

- Discovered in Epic 5: ` → `[ACTION]` | **high** | S05.01–S05.05 ATDD checklists never created (5/12 stories) · `IMPACT: standards_update` |

## Anti-Patterns

- Discovered in Epic 5: ` → `[ACTION]` | **medium** | OBS-001 circuit breaker 4xx miscounting — same bug in E04 + E05 files · `IMPACT: prompt_adjustment` |

## Anti-Patterns

- Discovered in Epic 5: ` → `[ACTION]` | **medium** | No k6 throughput baseline — 3rd consecutive epic deferral · `IMPACT: story_injection` |

## Patterns

- Discovered in Epic 6: TierGate as FastAPI per-route `Depends()` injection (never middleware) prevents all tier bypass vectors — field-level Pydantic model enforcement (`OpportunityFreeResponse` with exactly 6 fields) verified at model level by ATDD

## Patterns

- Discovered in Epic 6: Atomic Redis Lua script (`_USAGE_LUA` module-level constant) is the mandatory pattern for all metering operations — GET + conditional INCR + EXPIRE in single script; testcontainers Redis required for concurrency test (not fakeredis)

## Patterns

- Discovered in Epic 6: Negative ATDD assertions prevent silent architectural mistakes in streaming stories — assert `new EventSource` NOT present, `Axios` NOT used for streaming body; these are class-of-bug invisible at runtime

## Patterns

- Discovered in Epic 6: URL-driven filter/sort/pagination state is mandatory in frontend — `useSearchParams()` + `router.replace()`; stale-closure prevention via `useRef(onChange)`; zero `useState` for URL-representable state

## Patterns

- Discovered in Epic 6: Native `fetch` + `ReadableStream` for SSE streaming POST — `EventSource` is GET-only and silently returns cached data; `Axios` buffers body; only native `fetch` supports streaming POST response bodies

## Patterns

- Discovered in Epic 6: Dual-session service functions span pipeline/client schemas — separate `MetaData(schema=...)` instances per schema; `get_pipeline_readonly_session` for reads, `get_db_session` for writes; no cross-schema FK

## Patterns

- Discovered in Epic 6: `cancelledIdsRef` (`useRef<Set<string>>`) checked at every async boundary in multi-step upload flows — prevents cancelled-but-continuing uploads from mutating component state after abort

## Patterns

- Discovered in Epic 6: Over-fetch + scope-filter pattern for related items — fetch N (e.g., 10), filter to max (e.g., 5) via `is_in_scope()`; avoids complex scoped-count SQL when result set is small

## Patterns

- Discovered in Epic 6: Global Axios interceptor receives `show` callback as parameter (not `useStore.getState()` directly) — prevents circular dependency between `packages/ui` and `apps/client`; always `return Promise.reject(error)` to preserve TanStack Query error states

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **critical** | SSE generator lifecycle bugs reach senior dev review — semaphore permit leak on timeout, quota incremented before 404 check, generator not closed in finally, no terminal event on upstream EOF · `IMPACT: prompt_adjustment` · Template: (1) quota check BEFORE StreamingResponse creation, (2) generator closed in finally, (3) terminal event guaranteed, (4) fresh session_factory inside generator body

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **high** | Cross-story query key mismatch — S06.13 referenced `["opportunity-detail", id]` but S06.11 defined `["opportunity", id]`; `invalidateQueries` silently fails; frontend story specs must declare upstream query keys as explicit dev constraints · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **critical** | Dependabot not configured — 4th consecutive epic carry-forward (E01→E02→E05→E06); unscanned pyclamd, boto3, fakeredis, testcontainers; Beta milestone blocker · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **high** | Prometheus /metrics not bootstrapped — 4th consecutive epic carry-forward; PRD p95 <200ms REST / <500ms SSE TTFB unverifiable; Demo milestone blocker · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **high** | k6 performance baseline absent — 4th consecutive epic deferral; PostgreSQL FTS under 10K+ opportunities unvalidated; SSE concurrency cap (10/pod) unvalidated · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **high** | S06.05 (Opportunity Detail API) not completed at sprint close — carry forward as P0 Sprint 7 story; blocks proposal context and frontend detail page API wiring · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **medium** | Story completion gate incomplete — stories marked `done` without all three gates satisfied: (1) senior dev review APPROVED, (2) ATDD tests GREEN, (3) i18n parity check passing · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **medium** | UsageGate Redis fail-open not specified — Redis outage blocks all paid-tier AI summary requests; same gap as E02 rate-limiter; add fail-open requirement to all Redis-gated story templates · `IMPACT: prompt_adjustment`

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **medium** | ClamAV timeout-to-failed transition out of scope — documents stuck in `pending` undownloadable indefinitely on ClamAV outage; inject Sprint 7 background job story · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 6: `[ACTION]` | **medium** | E06-P0-010 Playwright E2E deferred — compensated by 862+ Vitest component tests but cross-story UI tier enforcement not exercised end-to-end; inject Sprint 7 hardening task · `IMPACT: story_injection`

## Patterns

- Discovered in Epic 6: ` |

## Patterns

- Discovered in Epic 6: ` |

## Patterns

- Discovered in Epic 6: ` |

## Patterns

- Discovered in Epic 6: ` |

## Patterns

- Discovered in Epic 6: ` |

## Patterns

- Discovered in Epic 6: ` |

## Patterns

- Discovered in Epic 6: ` |

## Patterns

- Discovered in Epic 6: ` |

## Patterns

- Discovered in Epic 6: ` |

## Patterns

- Discovered in Epic 7: Security hardening story injection is the correct NFR FAIL response — when NFR assessment returns FAIL on security/maintainability, inject a P0/BLOCKER hardening story (S07.17 pattern); ACs derived directly from NFR finding items; resolves within same sprint

## Patterns

- Discovered in Epic 7: Content-hash optimistic locking is the standard for collaborative document editing — `content_hash` column (SHA-256), client sends hash in request body, `SELECT FOR UPDATE` + hash comparison in transaction, 409 Conflict with structured body on mismatch, frontend conflict resolution dialog; both backend (content save API) and frontend (conflict dialog) stories required to close R-001 class risks

## Patterns

- Discovered in Epic 7: Vitest source-inspection ATDD scales to complex multi-panel frontend components — regex/import assertions without runtime rendering; stable in CI; enables RED-phase tests before components exist; runtime RTL/JSDOM reserved for user-interaction flows; S07.15 (87 tests, all GREEN) is the reference implementation

## Patterns

- Discovered in Epic 7: ThreadPoolExecutor is mandatory for CPU-bound library calls in FastAPI — any endpoint calling WeasyPrint, python-docx, Pillow, lxml MUST use `await loop.run_in_executor(executor, fn, *args)`; event loop blocking is invisible in unit tests and only manifests under load; ATDD checklists for export/render stories must assert "function uses `run_in_executor`"

## Patterns

- Discovered in Epic 7: Reset-stuck background task is required for all long-running async state machines — any resource with an in-progress status (generation, scanning, export, crawler run) must have a Celery Beat cleanup task marking stale entries FAILED after configurable TTL; S07.05's `reset_stuck_proposals_task` is the reference implementation; add as required AC to all stories that introduce a `status` field with in-progress variant

## Patterns

- Discovered in Epic 7: TEA test review catches backend/frontend contract schema drift and spec-vs-implementation constant contradictions invisible to ATDD — S07.11 review (95/100) caught `ProposalResponse` missing `current_version_number`/`generation_status` and `useBreakpoint` 1280px vs AC5 1024px threshold; TEA review is not optional

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **critical** | TRACE_GATE FAIL for 17-story epic where test existence = 100% but implementation GREEN = 27.3% — gate does not distinguish test quality from implementation completeness; for epics >12 stories, need per-completed-story GREEN gate separate from overall coverage gate · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **critical** | R-001 (data loss via optimistic locking, Score 9) has implementation (SELECT FOR UPDATE) but 0% confirmed passing ATDD tests — highest-risk scenario unverified; inject Sprint 8 GREEN-phase verification story; block conflict dialog merge on this gate · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **critical** | k6 performance baseline absent — 5th consecutive epic carry-forward (E03→E05→E06→E07); SSE TTFB for AI draft generation and PDF export p95 unvalidated; hard block on Epic 8 kickoff · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **critical** | Dependabot not configured — 5th consecutive epic carry-forward (E01→E02→E05→E06→E07); E07 adds weasyprint, python-docx, tiptap/*, recharts, @dnd-kit/sortable unscanned; hard block on Epic 8 kickoff · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **high** | Frontend response types manually duplicated from backend Pydantic models cause schema drift — `ProposalResponse` missing `current_version_number` and `generation_status`; caught only in TEA review; frontend types MUST be generated via codegen (openapi-typescript) from backend OpenAPI spec; manual duplication prohibited · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **high** | AC numeric constants (breakpoint thresholds, timeouts, file size limits) absent from implementation task descriptions — 1024px vs 1280px mismatch only caught in TEA review; add "Implementation constants" subsection to each story AC listing all numeric thresholds · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **high** | TEA review coverage at 1/17 stories — 5th consecutive epic with systematic TEA under-execution; configure TEA review score ≥ 80/100 as blocking gate before story transitions review→done · `IMPACT: config_tuning`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **high** | P0 areas with 0% confirmed-passing tests (S07.10 export, S07.13 E2E SSE, S07.04 optimistic locking) — stories with P0 ACs must have at least 1 GREEN test per P0 scenario before TRACE_GATE runs; 0-GREEN P0 = story cannot be `done` · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **high** | Content block prompt injection sanitization deferred (E07-R-005) — S07.09 has no explicit sanitization layer; AI Gateway is necessary but not sufficient; inject Sprint 8 security story for body sanitization before persistence and before forwarding · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **medium** | ATDD checklist internal test count summary vs level-by-level breakdown mismatch (S07.17: 27 stated vs 39 counted) — second occurrence; add auto-computed total comment to checklist template; CI lint step validates summary total matches level-by-level breakdown · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 7: `[ACTION]` | **medium** | Security audit artifact and ATDD checklist describe same story in conflicting phases (S07.17: 41 GREEN in security audit vs 39 RED in ATDD checklist) — security audit must include "ATDD Overlap" section; ATDD checklist is always canonical phase indicator · `IMPACT: standards_update`

## Patterns

- Discovered in Epic 7: — 6 new patterns**

## Anti-Patterns

- Discovered in Epic 7: / [ACTION] — 9 new items**

## Patterns

- Discovered in Epic 8: Clean billing service boundary is the reference architecture for payment-adjacent services — `billing_service.py` (Stripe customer/subscription), `webhook_service.py` (idempotent event processing), `vies_service.py` (VAT/VIES with fallback), `tier_gate.py` (FastAPI `Depends()` gating), `tier_cache.py` (Redis + DB fallback), `api/v1/billing.py` (thin router); no cross-service DB joins; all events via Redis Streams; 100% isolated in `client-api/services/billing/`

## Patterns

- Discovered in Epic 8: VIES (and any external validation service on the registration critical path) must fail-open to `pending` status — SOAP 503/timeout → `vat_validation_status: pending`; never block registration; retry on next billing event; apply reverse-charge only on `status: valid`; S08.10's three GREEN integration tests (8.10-API-001/002/003) are the reference

## Patterns

- Discovered in Epic 8: Stripe customer provisioning (and any external account setup) must run as `BackgroundTask` after registration returns 201 — fail-open pattern; `stripe_customer_id` nullable until provisioned; all billing endpoints return 422 with structured error on missing customer; prevents external payment service outages from blocking user registration

## Patterns

- Discovered in Epic 8: Webhook dedup table with `event_id` unique constraint is the mandatory idempotency mechanism for all payment webhooks — INSERT INTO `webhook_events(stripe_event_id)` inside the processing transaction; unique-constraint violation → return 200 (already processed); NOT in-memory state or Redis TTL; reference: S08.04 `webhook_events` table

## Patterns

- Discovered in Epic 8: Tier cache invalidation must use DELETE (not SET) on subscription change events — DELETE forces the next request to read authoritative DB value and repopulate with correct tier; SET would require knowing new tier at consumer time (read-before-write race); 60s TTL then keeps fresh value warm; reference: S08.14 `tier_cache.delete(company_id)` pattern

## Patterns

- Discovered in Epic 8: `asyncio.to_thread()` is mandatory for synchronous I/O SDKs inside `async def` FastAPI handlers — Stripe Python SDK, VIES SOAP clients, and any synchronous HTTP/DB library MUST be wrapped with `asyncio.to_thread(sync_fn, *args)` to avoid blocking the event loop; analogous to E07 ThreadPoolExecutor requirement for CPU-bound rendering libraries

## Patterns

- Discovered in Epic 8: NFR PASS without hardening story injection signals security architecture maturity — billing epic correctly applied E07 S07.17 lessons (HMAC, idempotency, GDPR data processing) from the start; when security/NFR mitigations are designed-in from story ACs rather than retrofitted, NFR assessment produces PASS (with CONCERNS) instead of FAIL→injection cycle

## Anti-Patterns

- Discovered in Epic 8: `[ACTION]` | **critical** | Retrospective [ACTION] items not executed in subsequent epic — k6 baseline, Dependabot, and R-001 optimistic locking GREEN verification were designated Sprint 8 hard deliverables in E07 retro; none evidenced complete in E08; retro-to-action feedback loop is broken; Orchestrator ChangeEvaluator must create verification tasks for all SEVERITY:critical [ACTION] items and check them at the next retrospective before new items are accepted · `IMPACT: config_tuning`

## Anti-Patterns

- Discovered in Epic 8: `[ACTION]` | **critical** | k6 performance baseline absent — 6th consecutive epic carry-forward (E03→E05→E06→E07→E08); `load-test-results.md` remains an unfilled template; Redis usage metering 10K concurrent INCR (8.8-PERF-001) never written; PRD p95 SLA unverifiable for any service; gate S12.17 delivery on k6 scripts written AND executed AND committed · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 8: `[ACTION]` | **critical** | Dependabot not configured — 6th consecutive epic carry-forward (E01→E02→E05→E06→E07→E08); E08 adds Stripe SDK, Stripe.js, VIES client unscanned; single `.github/dependabot.yml` file, <30 minutes; block Epic 9 kickoff on this being merged · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 8: `[ACTION]` | **high** | P0 E2E specs with ≥80% `test.skip` are NONE coverage (not PARTIAL) — `billing-checkout.spec.ts` and `billing-vat.spec.ts` are inert Playwright specs that consumed engineering time without providing quality assurance; classify as NONE in traceability matrix; ATDD checklists must include E2E test activation as a blocking sub-task before story is marked `done` · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 8: `[ACTION]` | **high** | Stripe outbound circuit-breaker absent — E04 two-layer resilience pattern (`circuit_breaker(retry(http_factory))`) not extended to `billing_service.py` / `vies_service.py`; Stripe 5xx/timeout fails the request with a logged error only; no circuit-breaker opens to protect a degraded Stripe endpoint; all outbound payment SDK calls must adopt the E04 pattern · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 8: `[ACTION]` | **high** | No Prometheus billing metrics — revenue-critical path completely unobservable; missing: webhook processing latency histogram, `billing_usage_sync_drift_total` gauge, Stripe API error counter, active tier distribution gauge, trial-to-paid conversion counter; inject 5 metrics into S12.17 before GA · `IMPACT: story_injection`

## Anti-Patterns

- Discovered in Epic 8: `[ACTION]` | **high** | TEA test review absent for all 14 stories — 6th consecutive epic with zero TEA review coverage; S08.04 (webhook), S08.08 (usage metering), S08.10 (VAT/VIES) were prime review candidates; TEA review must gate `review → done` story transition; minimum 3 scored reviews (≥80/100) per epic · `IMPACT: config_tuning`

## Anti-Patterns

- Discovered in Epic 8: `[ACTION]` | **high** | `invoice.payment_failed → past_due` P0 scenario unit-only — no integration test fires mock `invoice.payment_failed` through webhook endpoint and asserts DB status=past_due; revenue-critical path: failing invoices that don't downgrade = unpaid access; add integration test case to `test_stripe_webhook_flow.py` · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 8: `[ACTION]` | **high** | TRACE_GATE 80% overall threshold is incompatible with TDD RED-phase development — 2nd consecutive FAIL (E07+E08) with 100% test existence; gate penalises TDD methodology compliance; revise: primary gate = each `done` story has ≥1 GREEN test per P0 AC; secondary = overall FULL% (informational during sprint) · `IMPACT: standards_update`

## Anti-Patterns

- Discovered in Epic 8: — 8 new patterns, 9 new [ACTION] items**

## Patterns

- Discovered in Epic 9: ` | Dual-layer idempotency: DB constraint + Redis SETNX | — | — |

## Patterns

- Discovered in Epic 9: ` | `asyncio.to_thread` for Celery `.delay()` in async consumers | — | — |

## Patterns

- Discovered in Epic 9: ` | 404 (not 403) for cross-user scoped resource access | — | — |

## Patterns

- Discovered in Epic 9: ` | Narrow except clauses — never swallow `celery.exceptions.Retry` | — | — |

## Patterns

- Discovered in Epic 9: ` | Fernet encryption canonical module for OAuth tokens | — | — |

## Patterns

- Discovered in Epic 9: ` | Stripe counter: read → API(idempotency-key) → GETDEL | — | — |

## Patterns

- Discovered in Epic 9: ` | ECDSA webhook validation on raw body, fail-closed | — | — |

## Patterns

- Discovered in Epic 9: ` | Beat schedule constants cross-checked vs spec before story creation | — | — |

## Anti-Patterns

- Discovered in Epic 9: ` | Story file left at pre-dev status after completion (RC2 — 2nd recurrence) | — | — |

## Anti-Patterns

- Discovered in Epic 9: ` | Frontend component test deferral without activation story | — | — |

## Anti-Patterns

- Discovered in Epic 9: ` | Shipping new long-running service without Prometheus metrics | — | — |

## Anti-Patterns

- Discovered in Epic 9: ` | `epic-N: in-progress` not closed when all stories done (4th recurrence) | — | — |

## Anti-Patterns

- Discovered in Epic 10: Missing Epic-Level Test Design:** No `test-design-epic-10.md` exists, leaving NFR verification (Performance, Scalability) undefined.

## Anti-Patterns

- Discovered in Epic 10: Security Isolation Gap:** Delayed S10.02 middleware leaves proposal workspace endpoints exposed to UUID-based existence leaks.

## Anti-Patterns

- Discovered in Epic 10: TEA Coverage Gap:** 0 of 16 stories have TEA review scores, marking the 7th consecutive epic with this gap.

## Patterns

- Discovered in Epic 10: 100% RED-Phase Traceability:** All 14 Epic ACs are mapped to failing tests, providing a robust TDD foundation.

## Patterns

- Discovered in Epic 10: Audit & i18n Consistency:** 100% compliance on backend audit trail assertions and frontend EN/BG parity checks.

## Patterns

- Discovered in Epic 11: **Agent Integration Pattern (frozen standard)** — All agent calls use `AiGatewayClient.run_agent(agent_name, payload)`, flat 503 body `{"message": "...", "code": "AGENT_UNAVAILABLE"}`, `X-Caller-Service` header, configurable timeout, no client-side retry. S11.04 is the canonical backend reference; S11.09 is the canonical admin-api reference. Deviations block review approval.

## Patterns

- Discovered in Epic 11: **Null vs. Empty List for Optional Response Fields** — `null` signals field absence (UI suppresses component), `[]` signals presence with no content (UI shows empty state). This distinction must be preserved in AC, Pydantic schema, parser function, and frontend conditional render. Reference: S11.07 `gantt_data` graceful degradation.

## Patterns

- Discovered in Epic 11: **Migration Downgrade Multi-Row Safety** — Any migration downgrade that recreates a UNIQUE or PK constraint must include a regression test inserting ≥2 rows under the same grouping key before downgrading. Reference: S11.01 `ROW_NUMBER() OVER (PARTITION BY company_id)` pattern.

## Patterns

- Discovered in Epic 11: **Arithmetic Tolerance Constant** — Float comparisons in financial/budget calculations use a module-level `_ARITHMETIC_TOLERANCE = 0.01` constant. EU grant budgets don't go below half-euro; 1-cent tolerance avoids float precision failures. Reference: S11.05.

## Anti-Patterns

- Discovered in Epic 11: **[ANTI-PATTERN] "Done = Stub"** — [ACTION] `IMPACT: standards_update` `SEVERITY: critical` — Story marked `done` with frontend using stub API functions and zero GREEN backend tests for covered ACs. S11.07 TRACE_GATE FAIL (AC4/AC5 GAP) is the instance. Done gate must verify: (1) no stub references for story-owned endpoints, (2) ≥1 GREEN test per covered AC.

## Anti-Patterns

- Discovered in Epic 11: **[ANTI-PATTERN] Hardcoded English Strings in TSX Fix Passes** — [ACTION] `IMPACT: standards_update` `SEVERITY: high` — Developers add i18n keys for AC-specified strings but add literal English for strings introduced during fix passes (toast messages, placeholders, validation messages). Occurred in S11.11, S11.13, S11.14 across 3 review rounds. Fix: `no-literal-text` ESLint rule for `apps/client` and `apps/admin` — lint error, not review finding.

## Anti-Patterns

- Discovered in Epic 11: **[ANTI-PATTERN] Design System Component Violations Not Caught at ATDD Level** — [ACTION] `IMPACT: prompt_adjustment` `SEVERITY: high` — `<select>` (native) used instead of `<Select>` from `@eusolicit/ui` in two panels (S11.12). AC specified design system component. ATDD source-inspection assertion catches this at RED phase: `assert "<Select" in component_source`. Add to frontend story template for all `<Select>`, `<Dialog>`, `<Sheet>`, `<Tabs>` usages.

## Anti-Patterns

- Discovered in Epic 11: **[ANTI-PATTERN] Retry Buttons Bypassing Form Validation** — [ACTION] `IMPACT: prompt_adjustment` `SEVERITY: high` — Three panels in S11.12 called `mutation.mutate()` directly on retry, bypassing Zod validation. Retry must always route through `form.handleSubmit()` or add explicit pre-mutate validation guards. Add as explicit AC requirement: "Retry re-validates via `form.handleSubmit()`."

## Anti-Patterns

- Discovered in Epic 11: **[ANTI-PATTERN] TEA Reviews in Separate Injected Stories** — [ACTION] `IMPACT: config_tuning` `SEVERITY: critical` — `inj-03-tea-review-backlog` has been `ready-for-dev` for 9 consecutive epics; never executed. Separate stories get pre-empted. Fix: TEA review must be an AC on each story's done transition, not a standalone story.

## Anti-Patterns

- Discovered in Epic 11: **[ANTI-PATTERN] 503 Error Body Assertion Checking Wrong Shape** — [ACTION] `IMPACT: standards_update` `SEVERITY: high` — S11.03 Pass 3: tests asserted `data["detail"][...]` for 503 responses but router returns flat `{"message", "code"}` via JSONResponse. Template: `assert "message" in data and "code" in data and "detail" not in data` for all agent-unavailable tests.

## Anti-Patterns

- Discovered in Epic 11: **[ANTI-PATTERN] Null JSONB List Field Patch Semantics Undefined** — [ACTION] `IMPACT: prompt_adjustment` `SEVERITY: medium` — `PATCH {"rules": null}` returned 500 in S11.08 round 1. All PATCH endpoints with nullable list fields must include explicit AC: `PATCH with null for list field → 200, field becomes []` (clear semantics). Add to admin CRUD story template.

## Patterns

- Discovered in Epic 3: ** 7 positive patterns confirmed and codified (QueryGuard, useZodForm+FormField, dual-layer auth guard, packages/ui as platform library, TRACE_GATE loop-closure, server-compatible shell wrappers, deferred-work tracking)

## Anti-Patterns

- Discovered in Epic 3: ** 7 anti-patterns captured, including 2 **new** additions to `project-context.md`:

## Patterns

- Discovered in Epic 4: `/`[ANTI-PATTERN]` + `IMPACT`/`SEVERITY` tags

## Anti-Patterns

- Discovered in Epic 4: SEVERITY: high`

## Anti-Patterns

- Discovered in Epic 4: ` + `IMPACT`/`SEVERITY` tags

## Patterns

- Discovered in Epic 5: | Severity |

## Anti-Patterns

- Discovered in Epic 5: | Severity |

## Patterns

- Discovered in Epic 9: /[ANTI-PATTERN]/[ACTION] findings are in the retro artifact for the project-context merge. Good work shipping a complex notification stack, Deb. Epic 10 is waiting — and this time, we run k6 *first*."

## Anti-Patterns

- Discovered in Epic 9: /[ACTION] findings are in the retro artifact for the project-context merge. Good work shipping a complex notification stack, Deb. Epic 10 is waiting — and this time, we run k6 *first*."

## Patterns

- Discovered in Epic 13: **NFR hot-fix + proper story workflow** — Emergency inline commit unblocks NFR halt same day; proper hardening story created in parallel for full spec implementation. Two-step pattern: (1) minimal inline patch to restore NFR PASS, (2) proper story for refactor+tests+audit. Reference: S07.17 `2d41fcf` + story 7-17.

## Patterns

- Discovered in Epic 13: **`asyncio.CancelledError` requires explicit handler in Python 3.8+** — `CancelledError` inherits from `BaseException`, not `Exception`. Bare `except Exception` silently drops cancel signals in coroutines. Every `async def` with `async with / try / finally` blocks must include `except asyncio.CancelledError: raise` (or handle explicitly). This is invisible in eager-mode tests.

## Patterns

- Discovered in Epic 13: **structlog stdlib routing required for pytest caplog** — `structlog.PrintLoggerFactory()` bypasses stdlib logging; pytest `caplog` only captures stdlib records. Fix: session-scoped autouse fixture `configure_structlog_stdlib` in service `conftest.py` routes structlog through `stdlib.LoggerFactory()`. Required in every service's test suite.

## Patterns

- Discovered in Epic 13: **Fire-and-forget audit write canonical form (Rule 45)** — `asyncio.create_task(_write_generation_audit(...))` scheduled from `_run_generation_task.finally`. Task owns its own `async with session.begin()`, catches all exceptions, logs `audit_write_failed` at ERROR. NEVER `await write_audit_entry()` + `await session.commit()` inline on the TTFB path. Canonical form established in S07.17 review follow-up.

## Patterns

- Discovered in Epic 13: **Single-point SSE error sanitization helper** — All SSE error frames for a given endpoint MUST flow through one `_sanitize_error(exc, *, correlation_id, ...)` helper. No inline `{"error": ..., "code": ...}` construction outside the helper. Helper emits stable machine code + correlation_id; never raw exception fields. Reference: `_sanitize_generation_error()` in `proposal_service.py`.

## Patterns

- Discovered in Epic 13: **Coordinator story pattern for multi-concern hardening epics** — A coordinator story owns end-to-end AC acceptance + close-out gate; narrow sub-stories deliver independent slices. Coordinator cannot be `done` until all sub-story ACs are GREEN. Prevents mega-story scope explosion. Reference: `drift-recovery-story` + `inj-01/02/03` + `dw-01/02/03` in E13.

## Patterns

- Discovered in Epic 13: **Hot-Fix Context section in story Dev Notes** — Any story building on a prior emergency fix must include a "Hot-Fix Context" section: exact commit hash, which files were changed, what was done inline vs. what remains for the story. Prevents dev agents from re-implementing already-merged changes. Reference: S07.17 "Hot-Fix Context (Pre-Applied — Verify, Don't Re-Implement)".

## Anti-Patterns

- Discovered in Epic 13: **[ANTI-PATTERN] Dev session regression overwrites approved code** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `critical` — A new dev session modified an endpoint (proposals.py) for an unrelated AC (AC18 sibling story), introducing a duplicate audit-row pattern that contradicted the 2nd-pass approved code. Root cause: the agent did not re-read prior resolved findings before editing shared files. Fix: dev-story template must include: "Before editing any file with prior resolved review findings, re-read the file and list all resolved deviations to preserve." Reference: S07.17 F10/F11/F12, 3rd review pass 2026-04-23.

## Anti-Patterns

- Discovered in Epic 13: **[ANTI-PATTERN] Review approval without quoted test execution output** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `critical` — 2nd-pass adversarial review approved Story 7-17 claiming all tests passing; 3rd review found 3 integration tests failing at HEAD. The 2nd-pass reviewer trusted the agent's summary instead of re-executing. Fix: code-review template MUST require a quoted pytest output line (e.g., `3 failed, 16 passed`) for any "all tests passing" claim. Approval without a quoted execution result is invalid. Reference: S07.17 3rd review, 2026-04-23.

## Anti-Patterns

- Discovered in Epic 13: **[ANTI-PATTERN] sprint-status `done` set without cross-checking story file status** — `[ACTION]` IMPACT: `config_tuning` SEVERITY: `high` — Story 7-17 shows `done` in sprint-status.yaml but `review` in story file with 3 blocking findings. False completion signal to orchestrator and retrospective tooling. Fix: orchestrator story-close workflow must verify story file status = `done` before writing `done` to sprint-status. Reference: 7-17 discrepancy at E13 retro time.

## Anti-Patterns

- Discovered in Epic 13: **[ANTI-PATTERN] AC text not updated to reflect implementation-time schema choices** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `medium` — AC1 still specifies `{"error":"ai_gateway_error","correlation_id":"<uuid>"}` but implementation emits `{"error": "<prose>", "code": "gateway_error", "correlation_id": "<uuid>"}`. Future reviewers will rediscover as spec violation. Fix: any implementation-time deviation from an AC schema must update the AC text before review approves. AC text is the contract. Reference: S07.17 Known Deviations, 2026-04-18 / 2026-04-23.

## Anti-Patterns

- Discovered in Epic 13: **[ANTI-PATTERN] Injected carry-forward stories deprioritized behind new feature work** — `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — Epic 13 was created to close 6+ carry-forwards (Dependabot, k6, TEA reviews, Stripe CB, billing metrics). At partial retrospective time, 0/6 carry-forwards are closed; all 7 non-7-17 stories remain at `ready-for-dev`. Injected `inj-*` stories must be set as p0 priority and execute BEFORE any new feature stories. Carry-forward accumulation → orchestrator config change required. Reference: E13 partial retro 2026-04-25.

## Anti-Patterns

- Discovered in Epic 13: **[ANTI-PATTERN] Pydantic response schema uses bare `str` for enumerable status fields** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `medium` — `ProposalDetailResponse.status` was bare `str` rather than `Literal["draft", "active", "archived"]`. Prevents OpenAPI enum schema + allows unknown values. Fix: all Pydantic response schemas with known value sets MUST use `Literal[...]` or `StrEnum`. Add to backend story template. Reference: S07.17 DW-01 AC5.

## Anti-Patterns

- Discovered in Epic 13: **[ANTI-PATTERN] `asyncio.get_event_loop()` in Celery tasks (deprecated Python 3.12)** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `medium` — DW-02 documents `asyncio.get_event_loop()` deprecated in Python 3.12; Celery sync→async bridge must use `asyncio.new_event_loop()` in `try/finally loop.close()`. Never `asyncio.run()` (fails with running loop). Add to Celery task template. Reference: DW-02 Dev Notes.

## Patterns

- Discovered in Epic 14: **Carry-forward codification in Dev Notes** — A story's Dev Notes should explicitly list anti-patterns from earlier stories in the same epic by story number (e.g. "Epic 14.2 BLOCKING #3 — do not rebuild bespoke test fixtures"). This reduces repeat violations within an epic. Reference: S14.04 Dev Notes §Previous Story Learnings.

## Patterns

- Discovered in Epic 14: **Partial-unique WHERE predicate for soft-delete–aware uniqueness** — `UNIQUE (col1, col2) WHERE archived_at IS NULL` (or `WHERE accepted_at IS NULL`) is the correct pattern for logical-delete tables. `IntegrityError → 409` translation in the service layer prevents 500s on duplicate active records. Reference: migration 046 `ix_client_workspaces_company_name_active`, migration 048 `ix_external_collaborators_active_email`.

## Patterns

- Discovered in Epic 14: **External JWT claim-shape isolation** — External collaborator tokens must omit `company_id` and `subscription_tier` claims so that `get_current_user` rejects them with a `KeyError → 401` before any auth logic runs. Define `ExternalCollaboratorPrincipal` as a separate dataclass (non-subclass of `CurrentUser`) so the type system enforces non-substitutability. Endpoints that accept both use `Union[CurrentUser, ExternalCollaboratorPrincipal]` explicitly. Reference: S14.04 `core/security.py`.

## Patterns

- Discovered in Epic 14: **Single-use semantics via `accepted_at` DB stamp (extends E08 webhook dedup pattern)** — `accepted_at IS NOT NULL → 410 Gone` on subsequent magic-link accept calls. 410 is semantically correct (resource "gone" after consumption) vs 401 (unauthenticated) or 409 (conflict). Pattern mirrors `webhook_events.event_id` unique-constraint idempotency from E08. Reference: S14.04 AC 4, `external_collaborator_service.accept()`.

## Patterns

- Discovered in Epic 14: **Stripe seat isolation contract enforced at table level** — `count_active_seats()` MUST only query `company_memberships`, never `external_collaborators`. The two tables are separate by design (FR8.5: external collaborators do not consume paid seats). A dedicated integration test + Stripe-mock test create a regression guard. This "table-as-contract" pattern prevents billing drift from future schema changes. Reference: S14.04 AC 9, `billing_service.count_active_seats()`.

## Patterns

- Discovered in Epic 14: **Workspace re-validation on every external read** — A helper (e.g. `_verify_external_proposal_match`) re-checks `proposal.workspace_id == principal.workspace_id` on each request with an external token. If a proposal is administratively re-assigned, the old token immediately returns 404 (existence-leakage protection) rather than granting stale access. Reference: S14.04 `core/rbac.py`.

## Anti-Patterns

- Discovered in Epic 14: **[ANTI-PATTERN] Bespoke test fixtures rebuilt despite canonical helpers existing** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `critical` — EVERY story in Epic 14 was flagged for rebuilding `_register_and_verify_with_role`, `ASGITransport(app=fastapi_app)`, or `db_session` (no rollback) instead of using root `conftest.py` helpers. Fix: dev-story template MUST embed the canonical fixture import block (`from eusolicit_test_utils import register_and_verify_with_role, create_company_pair`); non-use = BLOCKING code review finding. Reference: S14.01 review, S14.02 BLOCKING #3, S14.04 Dev Notes anti-pattern #1.

## Anti-Patterns

- Discovered in Epic 14: **[ANTI-PATTERN] Story file `Status: review` while sprint-status says `done` (4/5 stories, 3rd consecutive epic)** — `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — The orchestrator story-close workflow MUST assert `grep "^Status: done" {story_file}` returns true before writing `done` to sprint-status.yaml. A story with `Status: review` in its file cannot be `done` in the pipeline. Reference: E14 retro 2026-04-27, also E12 and E13 retrospectives.

## Anti-Patterns

- Discovered in Epic 14: **[ANTI-PATTERN] Documentation-vs-code mismatch — claimed fix or test not in code on disk** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `critical` — S14.00 Round 3 claimed CTE tie-breaking fix applied but SQL was unchanged. S14.02 claimed 60-case `test_rbac_permission_matrix` but traceability scan found only 2 tests on disk. Fix: (1) code review MUST `grep` for every file in Dev Agent Record File List; (2) completion notes claiming a test exists MUST include verbatim `pytest -k <name> -v` output for that specific test. Reference: S14.00 Round 3–4, traceability-matrix.md E14 CONCERNS.

## Anti-Patterns

- Discovered in Epic 14: **[ANTI-PATTERN] Breaking API change on shared schema without full-suite validation** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `high` — S14.02 made `ProposalCreateRequest.workspace_id` required without updating existing callers; the in-story 5-test suite did not surface the regression. Fix: dev-story template gate: "If you modified any file in `schemas/`, paste the full `make test-service SVC=<svc>` output before requesting review." Reference: S14.02 BLOCKING #1.

## Anti-Patterns

- Discovered in Epic 14: **[ANTI-PATTERN] RBAC permission matrix claimed but absent from disk — critical security gap** — `[ACTION]` IMPACT: `story_injection` SEVERITY: `critical` — S14.02's 60-case parametrized RBAC matrix was reverted or overwritten during S14.03 scope-creep revert; only 2 tests remain in `test_workspace_rbac.py`. Multi-tenant workspace isolation is under-tested. Inject a P0 story to restore the full matrix before E15 ships workspace-dependent features. Reference: traceability-matrix.md Epic 14 CONCERNS, E14 retro 2026-04-27.

## Anti-Patterns

- Discovered in Epic 14: **[ANTI-PATTERN] Review-fix pass introduces new BLOCKING by deleting out-of-scope files** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `medium` — S14.03 review-fix deleted `workspace_service.py` and `test_workspace_rbac.py` as "scope creep revert", breaking `client-api` startup with import errors. Fix: review-fix passes MUST run import smoke test (`python -c "from <service>.main import app"`) before claiming complete. Add to dev-story template as a mandatory gate on fix passes. Reference: S14.03 Round 2 BLOCKING findings.

## Anti-Patterns

- Discovered in Epic 14: **[ANTI-PATTERN] Playwright E2E marked done without execution due to missing system library** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `medium` — S14.03 E2E spec is "structurally correct and TypeScript-clean" but never ran (missing `libnspr4` in autopilot environment). Done gate for E2E ACs requires EITHER a quoted `N passed` Playwright summary OR an explicit "deferred to CI" deviation note citing the environment constraint. Reference: S14.03 Round 3 completion notes.

## Anti-Patterns

- Discovered in Epic 14: **[ANTI-PATTERN] TEA review absent for 10th consecutive epic (E04–E14)** — `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — `inj-03-tea-review-backlog` has been `ready-for-dev` for 10 consecutive epics and is never executed as a standalone story. Fix: TEA review is a story AC (not a separate story): dev agent runs `bmad-tea` before senior code review; score ≥ 80/100 required. Remove `inj-03` from backlog; embed TEA as a step in the dev-story prompt template. Reference: E14 retro 2026-04-27, E13/E11 action items.

## Anti-Patterns

- Discovered in Epic 14: **[ANTI-PATTERN] No NFR assessment for Epic 14; E13 HALT conditions (k6, Dependabot, billing CB) still unresolved** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `high` — No `bmad-testarch-nfr` was run for E14. The 4 NFR HALT conditions from E13 (k6 absent, OWASP/Dependabot absent, billing circuit-breaker absent, 7-17 completion integrity) remain open. Epics 15+ cannot proceed to production gate without resolving these. Epic 16 kickoff gated on: Dependabot merged + k6 written + billing CB implemented. Reference: test_artifacts/nfr-report.md (E13 FAIL), E14 retro 2026-04-27.

## Patterns

- Discovered in Epic 14: ` + 9 new `[ANTI-PATTERN]` entries

## Anti-Patterns

- Discovered in Epic 14: ` entries

## Patterns

- Discovered in Epic 15: **Lua atomicity proof via `asyncio.gather(10_000)`** — `test_concurrent_check_and_increment_enterprise` drives 10,000 concurrent `check_and_increment` calls and asserts `len(set(results)) == 10_000`. This is a sharper assertion than `max == 10_000` — it proves every counter value from 1 to 10,000 was assigned exactly once. Reference pattern for all Redis counter atomicity proofs. Reference: `test_usage_metering_bypass.py`.

## Patterns

- Discovered in Epic 15: **Workspace-scoped Redis key dual-lookup (scoped_id = workspace_id or company_id)** — Any Redis key that must be scoped to either workspace or company must: (a) check the workspace-scoped key first (`{workspace_id}:{resource_id}`), fall back to company-scoped (`{company_id}:{resource_id}`). This pattern is now consistent across: webhook write (`webhook_service.py:699-700`), usage gate bypass check (`usage_gate.py:196-219`), and per-bid status read. Asymmetric keys are the source of silent bypass failures.

## Patterns

- Discovered in Epic 15: **Explicit tier branch over implicit fall-through** — When a new tier is inserted between existing tiers in an `if/elif` or `match` chain, every existing fall-through default that implicitly assigns behaviour MUST get an explicit branch with a comment. Silent fall-through (`pro_plus` getting `enterprise`-equivalent scope by accident) is a latent bug. Reference: `opportunity_tier_gate.is_in_scope` lines 155–161 explicit Pro+ branch.

## Patterns

- Discovered in Epic 15: **`_*_TIERS_LIST` as explicit literal, not `list(frozenset)`** — Lists derived from frozensets for use in WHERE clauses, display, or ordering MUST be explicit literals (`["pro_plus", "enterprise"]`). `list(frozenset)` produces non-deterministic order across Python versions and runs, causing intermittent test failures and non-deterministic SQL IN() ordering. Reference: `tier_gate.py::_PRO_PLUS_TIERS_LIST`.

## Anti-Patterns

- Discovered in Epic 15: **[ANTI-PATTERN] Story marked `done` in sprint-status without code review Approve — worst instance to date** — `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — S15.1 is `done` in sprint-status with "Changes Requested" verdict unresolved and 8 BLOCKING findings. The orchestrator story-close workflow MUST verify: (1) story file `Status: done`, AND (2) Senior Developer Review section contains `REVIEW: Approve`. Both conditions required before sprint-status update. Reference: E15 retro 2026-04-27, sprint-status 15-1.

## Anti-Patterns

- Discovered in Epic 15: **[ANTI-PATTERN] FE↔BE request/response contract untested at integration level** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `critical` — S15.1 `PerBidCheckoutRequest.tier_id` vs AC/frontend `pricing_tier_id` was only caught by human code review. Backend integration tests locked in the wrong field name. Any story introducing a new POST body or response schema shared between FE and BE MUST include a contract test that sends the AC-specified field names and asserts the AC-specified response shape. AC field names are the contract, not the implementation.

## Anti-Patterns

- Discovered in Epic 15: **[ANTI-PATTERN] Migration seed rows not validated against AC spec table** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `critical` — S15.1 migration 050 seeded wrong BG labels, empty `feature_stack`, and non-null fake `stripe_price_id` despite the AC specifying exact column values. The migration idempotency test (AC 15 pattern) must assert each seeded column value against the AC spec, not just row count. A row count of 3 with wrong column values is a production bug.

## Anti-Patterns

- Discovered in Epic 15: **[ANTI-PATTERN] Locale and workspace_id hardcoded in Stripe success/cancel URLs** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `high` — `billing_service.create_per_bid_checkout_session()` used `/bg/workspace/default/` in success_url. Any service function building a frontend URL MUST accept `locale: str` and `workspace_id: UUID | str` as parameters. Template: `f"{settings.frontend_url}/{locale}/workspace/{workspace_id}/..."`. Hardcoded locale/workspace routes fail for all non-BG users and non-default workspaces. Reference: S15.1 B8.

## Anti-Patterns

- Discovered in Epic 15: **[ANTI-PATTERN] Frontend wired to non-existent backend endpoint** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `high` — S15.1 AC 10 frontend called `GET /api/v1/billing/per-bid/status` which has no backend implementation. Frontend stories must explicitly list each API endpoint they call, with a corresponding backend story or AC that creates it. If the endpoint is out of scope, use a local cache signal (query param from Stripe redirect) rather than a polling endpoint. Reference: S15.1 B7.

## Anti-Patterns

- Discovered in Epic 15: **[ANTI-PATTERN] FastAPI route handler opens own Redis connection instead of `get_redis_client` dep** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `high` — `admin_api/api/v1/pricing_tiers.py::_invalidate_pricing_tier_cache` used `aioredis.from_url(os.getenv(...))` bypassing the registered `get_redis_client` dependency. All Redis access in FastAPI handlers MUST go through `Depends(get_redis_client)` — for testability via override and connection pool sharing. Direct connection creation in route handlers is a BLOCKING code review finding. Reference: S15.1 M4.

## Anti-Patterns

- Discovered in Epic 15: **[ANTI-PATTERN] Two sources of truth for tier feature limits (M5 deferred from S15.0 review)** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `medium` — `usage_gate.FEATURE_TIER_LIMITS["ai_summary"]["professional"] = 50` vs `tier_access_policies.ai_summaries_limit = 100`. Two code paths compute the same limit from different values. Resolution: `usage_gate.FEATURE_TIER_LIMITS` should read from `tier_access_policies` at startup, OR a CI test must assert all limit values are identical across both sources. "Deferred to retrospective" is no longer an acceptable path for source-of-truth divergence — inject as a P1 carry-forward story immediately.

## Patterns

- Discovered in Epic 15: ` entries + 8 new `[ANTI-PATTERN]` entries appended

## Anti-Patterns

- Discovered in Epic 15: ` entries appended

## Anti-Patterns

- Discovered in Epic 15: ` `IMPACT: config_tuning` `SEVERITY: critical` — **Story marked done without code review approval.** S15.1 had "Changes Requested" with 8 BLOCKING findings; sprint-status was written to `done` anyway. Orchestrator story-close must verify *both* `Status: done` in story file AND `REVIEW: Approve` in review section.

## Anti-Patterns

- Discovered in Epic 15: ` `IMPACT: prompt_adjustment` `SEVERITY: critical` — **FE↔BE contract untested.** `tier_id` vs `pricing_tier_id` field name mismatch locked in by integration tests; every per-bid purchase would return 422 in production.

## Anti-Patterns

- Discovered in Epic 15: ` `IMPACT: prompt_adjustment` `SEVERITY: critical` — **Migration seed not validated against AC.** Wrong BG labels, empty `feature_stack`, fake non-null `stripe_price_id` seeded.

## Anti-Patterns

- Discovered in Epic 15: ` `IMPACT: standards_update` `SEVERITY: critical` — **Bespoke fixtures at 4th consecutive epic** despite explicit prohibition in the story's own Dev Notes.

## Patterns

- Discovered in Epic 17: **CRM Adapter Registry with `@register_adapter` decorator — stub-first extensibility.** `CRMAdapter` ABC in 17.0; `StubAdapter` registered per-provider; real adapters in 17.1/17.2/17.3 replace stubs via decorator without changing the dispatcher (`forward.py`). Structural test asserts every `ADAPTERS` entry declares `provider` + `rate_limit_config` and inherits `CRMAdapter`. Reference pattern for any new CRM/integration provider. Reference: 17.0 AC-3, 17.1 AC-1.

## Patterns

- Discovered in Epic 17: **Three-tier circuit-breaker hierarchy for CRM.** (1) `crm:{workspace_id}:{provider}` — per-workspace per-provider transient-failure breaker; (2) `crm:auth:{provider}` — provider-global OAuth client outage; (3) `crm:quota:{workspace_id}:salesforce` — Salesforce daily-quota exhaustion (independent from transient failures). Each breaker opens independently — W1 HubSpot outage does NOT affect W2. This is the reference for per-provider breaker scoping. Reference: 17.0 §4.2, 17.3 AC-1 §5.

## Patterns

- Discovered in Epic 17: **Tier-paused fail-CLOSED webhook contract.** Downgraded workspace returns `200 {"status": "tier_paused"}` (NOT 402) from inbound webhook handler; `sync_logs` row written with `auth_failed`/`tier_downgraded`; tier resolver defaults `None → 'starter'` on schema-availability outage (fail-CLOSED). Interactive connect endpoint uses 402; provider-facing webhook uses 200+tier_paused. Reference: 17.1 §8.12, 17.2 AC-6 §5.

## Patterns

- Discovered in Epic 17: **Runtime default-stage seed at OAuth callback inside SAVEPOINT — never in migration.** Default CRM stage mappings seeded per-workspace at OAuth callback time via `INSERT ... ON CONFLICT (workspace_id, provider, eu_solicit_status) DO NOTHING` inside `session.begin_nested()`. Migration creates table structure only. Re-connection is idempotent. Admin-api `reset-to-default` uses per-provider default-mapping registry dict. Reference: 17.1 AC-5, 17.2 AC-5, 17.3 AC-6.

## Patterns

- Discovered in Epic 17: **Cross-portal forged webhook detection must log at WARNING — silent drop hides hostile activity.** When a webhook's `portalId`/`company_id` matches W1's connection but the `deal_id` internal mapping belongs to W2, respond `200 "ignored"` AND emit structured `crm.webhook.cross_portal_attempt` WARNING log with `{attempted_portal, attempted_deal_id, owning_workspace}`. Pure `200 "ignored"` without the log masks cross-tenant probing. Reference: 17.1 AC-10 §1, 17.2 AC-10 §1.

## Patterns

- Discovered in Epic 17: **Salesforce daily-quota state machine with Redis-Lua midnight-UTC rollover.** `_SALESFORCE_QUOTA_LUA` is atomic: GET → threshold check → INCR+EXPIRE on `ok`/`warning`; NO INCR on `exhausted`. Key TTL = `seconds_until_next_midnight_UTC`. `crm:quota:{workspace_id}:salesforce` breaker `reset_timeout` matches same value. `Sforce-Limit-Info` header is observational only (org-wide); Redis counter is authoritative for our per-workspace usage. Reference: 17.3 AC-4.

## Patterns

- Discovered in Epic 17: **Provider OAuth credential styles differ — document with cross-references in adapter docstring.** HubSpot: form-encoded body credentials (NOT JSON, NOT Basic). Pipedrive: `Authorization: Basic base64(client_id:client_secret)` + form body (NOT body-credentials). Salesforce: form-encoded body with `client_id`+`client_secret` in body (NOT Basic, NOT JSON). Each adapter's `authenticate()` docstring must cross-reference the other providers' styles. Reference: 17.1 AC-2, 17.2 AC-2, 17.3 AC-2.

## Patterns

- Discovered in Epic 17: **Vendor-specific cursor datetime formats must be documented with cross-references.** HubSpot: epoch milliseconds (`hs_lastmodifieddate`). Pipedrive: UTC string `YYYY-MM-DD HH:MM:SS` (`update_time`). Salesforce ISO-8601+millis+TZ `2026-05-01T14:23:45.000Z` for SOQL (`LastModifiedDate`; normalize `+0000` → `+00:00` for Python `fromisoformat`). Future providers must document and cross-reference. Reference: 17.1 B-15, 17.2 §4.10, 17.3 AC-7.

## Patterns

- Discovered in Epic 17: **Webhook subscription lifecycle for subscription-model providers (Pipedrive pattern).** Provision `POST {api}/v1/webhooks` on OAuth callback; persist `provider_webhook_id`. Fire-and-forget `DELETE {api}/v1/webhooks/{id}` on `status → revoked`; log WARNING on failure (provider may already revoke server-side). Provision failure is graceful-degradation — connection committed, real-time updates unavailable until re-register. Reference: 17.2 AC-7.

## Anti-Patterns

- Discovered in Epic 17: **[ANTI-PATTERN] Story marked `done` in sprint-status with `Status: review` in story file — 12th occurrence** — `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — All 4 Epic 17 stories exhibit this. Orchestrator `2b-dev-story-verify` MUST verify both (1) `grep "^Status: done" {story_file}` is true AND (2) story file contains `REVIEW: Approve` before writing `done` to sprint-status.yaml. Reference: AP17-01, E15/E14/E13/E12 same action.

## Anti-Patterns

- Discovered in Epic 17: **[ANTI-PATTERN] 257 ATDD RED-phase tests shipped as `done` with 0 GREEN** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `critical` — Stories 17.1 (101), 17.2 (77), 17.3 (79) have all ATDD tests still `@pytest.mark.skip`. Dev agent implemented ACs and quoted passing pytest summaries but never returned to un-skip ATDD scaffolds. Fix: dev-story completion MUST quote ATDD un-skip count alongside pytest summary. A story with ATDD skip count > 0 cannot be marked `done`. Reference: AP17-02.

## Anti-Patterns

- Discovered in Epic 17: **[ANTI-PATTERN] k6 performance baseline absent — 11th consecutive epic; E16 hard gate ignored** — `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — E16 retro declared k6 baseline a hard gate for E17 kickoff. E17 proceeded anyway. Orchestrator `epic-kickoff` phase must verify `load-test-results.md` contains non-dash values before allowing first story `in-progress`. Reference: AP17-03, inj-02 (ready-for-dev since E13).

## Anti-Patterns

- Discovered in Epic 17: **[ANTI-PATTERN] TEA reviews absent for 12th consecutive epic** — `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — `inj-03-tea-review-backlog` has been `ready-for-dev` for 12 consecutive epics. Standalone story approach always fails. Fix: TEA score ≥ 80/100 is a required gate in `2b-dev-story-verify` — dev agent runs `bmad-tea` before senior code review. Remove `inj-03` from backlog. Reference: AP17-04, E14/E11 retro.

## Anti-Patterns

- Discovered in Epic 17: **[ANTI-PATTERN] Three consecutive BLOCKED verdicts without escalation** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `critical` — Story 17.0 received three consecutive `ARCHITECTURAL_DRIFT: blocking` verdicts before first resolve. Each false-positive HALT costs 4-6h. Fix: orchestrator must surface Telegram alert after 2 consecutive BLOCKED verdicts on same story; 3rd pass requires operator confirmation. Reference: AP17-05, E13 retro.

## Anti-Patterns

- Discovered in Epic 17: **[ANTI-PATTERN] Testcontainers Redis Lua atomicity deferred across three stories** — `[ACTION]` IMPACT: `standards_update` SEVERITY: `high` — 17.1 AC-8, 17.2 AC-8, 17.3 AC-4 all have `@pytest.mark.skip(reason='testcontainers infra deferred')` on Lua atomicity tests. The E15 `asyncio.gather(N)` proof pattern exists but was not applied to CRM rate-limit tests. Fix: add shared `testcontainers_redis` session-scoped fixture to integrations-api conftest; un-skip all Lua atomicity tests in E18.0. Reference: AP17-07, E15 `test_usage_metering_bypass.py`.

## Anti-Patterns

- Discovered in Epic 17: **[ANTI-PATTERN] Epic-level test design `test-design-epic-17.md` never created** — `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `medium` — All four ATDD checklists noted "No epic-level test design found". IR report (2026-04-28) flagged this gap. Fix: `test-design-epic-N.md` is a required deliverable in the `1-create-story` phase for the first story of each epic. Reference: AP17-08.

## Anti-Patterns

- Discovered in Epic 17: **[ANTI-PATTERN] Retro gate conditions serially ignored — gate enforcement is advisory not blocking** — `[ACTION]` IMPACT: `config_tuning` SEVERITY: `medium` — E16 retro identified 4 "Cannot proceed to E17 without" conditions; E17 proceeded without verifying any. Orchestrator `epic-kickoff` phase must parse the previous retro's critical actions and verify each before allowing first story `in-progress`. Reference: AP17-09.

## Patterns

- Discovered in Epic 18: **Trust Center multi-story chain (18-0→18-1→18-2) delivered without broken interfaces.** `compute_diff()` + `yaml_content_hash()` introduced in 18-0; reused without reimplementation in 18-1 (CI step) and 18-2 (publisher + SETNX idempotency key). `dpa_download_url` from 18-1 consumed by 18-2 email body with documented null-safe fallback (D-6). Anti-pattern fence rows #37 (no-reimplement-compute_diff) + #38 (no-cross-import-_PAID_TIERS) enforced the boundary. This is the reference design for multi-story epic chains. Reference: 18-0 AC-9 CR-6, 18-2 AC-1 §1, 18-2 D-6.

## Patterns

- Discovered in Epic 18: **`yaml_content_hash()` is more robust than git SHA as an idempotency key for CI-generated artifacts.** 8-char `sha256(yaml_bytes)[:8]` is deterministic across: run-locally → commit → push → CI re-run → whitespace edit. Git short-SHA is commit-cycle unstable. Any CI script that needs to avoid double-processing a re-run should key on content hash, not commit SHA. Downstream consumers: changelog prepend guard, SETNX `notification:dispatched:subprocessor:{changelog_sha}:{admin_user_id}`. Regression tests: `test_yaml_content_hash_is_stable_for_identical_bytes`, `test_idempotency_survives_commit_cycle_simulation`. Reference: 18-0 CR-6, 18-2 AC-10.

## Patterns

- Discovered in Epic 18: **`trust_render_allow_company_admin_fallback` defaulting False — privilege-adjacent features must opt-in, not opt-out.** When authority model is ambiguous (staff-admin allow-list vs. `is_company_admin`), the security-correct default is deny. Transition fallback is an explicit opt-in setting. This applies to any endpoint where the authorization model is being introduced or refined. Reference: 18-1 B1 Round 3.

## Patterns

- Discovered in Epic 18: **ACK-after-fanout + SETNX idempotency is the required pattern for all notification Redis Stream consumers.** Per-recipient `SETNX notification:dispatched:{event_type}:{content_hash}:{user_id}` TTL 86400; ACK only after all fan-out completes (claim-on-success spirit from S16 M6). Never ACK before fan-out — partial delivery creates silent data loss. Test: `test_ack_called_only_after_fanout_complete`. Reference: 18-2 AC-3, fence #40.

## Patterns

- Discovered in Epic 18: **locale defaults to `bg`, NOT `en`, for all SendGrid email deliveries in this platform.** `User.locale_preference` routed to SendGrid template selection; fallback = `bg`. `en` fallback would silently English-spam Bulgarian admin users. Anti-pattern fence #42. Any future notification story must honor this locale-router default. Reference: 18-2 AC-6, fence #42.

## Patterns

- Discovered in Epic 18: **Anti-pattern fence at 42 items — carry-forward and zero violations in E18.** All 42 fence rows were explicitly referenced and held across 18-0, 18-1, and 18-2 code reviews. No fence violations detected. The fence is growing correctly; each new story must import the full fence before adding net-new rows. Reference: 18-0 §4.6 (#1–#24), 18-1 §4.6 (#25–#36), 18-2 fence (#37–#42).

## Patterns

- Discovered in Epic 18: **Review pass efficiency: 2.3 passes/story average (E18) vs 4.5 passes/story (E17).** First-pass quality is improving. Contributing factors: (1) active anti-pattern fence reference, (2) explicit §4.7 inline test design with P0/P1 gating, (3) pre-review checklist completeness. The target is ≤2 passes/story. Continued improvement requires ATDD checklist artifacts in `test_artifacts/` for reviewer confidence.

## Anti-Patterns

- Discovered in Epic 18: **[ANTI-PATTERN] Story file `Status: review` when sprint-status is `done` — 13th occurrence (AP18-01).** `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — 18-0 and 18-1 still show `Status: review` in their story file headers despite sprint-status `done`. Only 18-2 reconciled correctly. Fix: the orchestrator `2b-dev-story-verify` phase must atomically patch `Status: done` in the story file at the same time it writes `done` to sprint-status. No additional manual step. Reference: AP17-C5 (13th occurrence).

## Anti-Patterns

- Discovered in Epic 18: **[ANTI-PATTERN] k6 performance baseline absent — 13th consecutive epic (AP18-02).** `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — `/trust` is public-facing and crawl-traffic exposed. `/api/v1/trust/artefacts/{slug}` triggers S3 head_object + presign on every request. Neither validated against PRD NFRs (p95 ≤ 200ms). E19 kickoff MUST be gated on non-dash values in `load-test-results.md`. Reference: `inj-02-k6-performance-baseline` (ready-for-dev since E13, 13th carry-forward).

## Anti-Patterns

- Discovered in Epic 18: **[ANTI-PATTERN] ATDD checklist artifacts missing for 18-1 and 18-2 — only 18-0 has a standalone `test_artifacts/` file (AP18-05).** `[ACTION]` IMPACT: `standards_update` SEVERITY: `high` — Stories with backend code (18-1 WeasyPrint/S3, 18-2 notification consumer) must produce `test_artifacts/atdd-checklist-{story-key}.md`. Inline §4.7 is a draft, not a substitute. Without standalone files, traceability matrix and TEA gate cannot reference them. Fix: promote §4.7 content to standalone files before story enters review. Reference: AP17-C2.

## Anti-Patterns

- Discovered in Epic 18: **[ANTI-PATTERN] WeasyPrint native library dependencies (Pango/Cairo) not in local test environment — 12 env-skipped tests (AP18-07).** `[ACTION]` IMPACT: `standards_update` SEVERITY: `medium` — Env skips look like failures and erode test suite confidence. Fix: `eusolicit-app/Makefile` `make install-native-test-deps` target + `packages/eusolicit-common/README.md` local setup instructions for `libpango-1.0-0`, `libcairo2`, `libgdk-pixbuf2.0-0`. Reference: 18-1 test suite, AP18-07.

## Anti-Patterns

- Discovered in Epic 18: **[ANTI-PATTERN] 18-1 Known Deviations D6–D11 (DoS surface + thread leak) deferred — hardening sprint never materialises (AP18-08).** `[ACTION]` IMPACT: `standards_update` SEVERITY: `high` — D6 (DoS on GET artefact, no rate-limit/timeout) and D7 (run_in_executor thread leak on POST timeout) are security/reliability concerns. E19-2 depends on `weasyprint_renderer`. Building the Outcome Brief PDF generator on a leaking foundation compounds risk. Fix: inject `18-3-trust-center-hardening` story with D6+D7 as P0 ACs before E19-2 dispatch. Reference: 18-1 Known Deviations D6/D7.

## Patterns

- Discovered in Epic 18: :**

## Patterns

- Discovered in Epic 20: **NEXT_PUBLIC_ as vendor public site-key — intentional browser exposure with documented threat model.** For third-party SDK site-keys that vendors designate as public (analogous to Google Analytics tracking ID), expose via `NEXT_PUBLIC_` prefix. Required documentation: (1) §6 Known Deviation with vendor threat model citation, (2) source-inspection test forbidding `eval(...)` in SDK init, (3) structlog scrub-keys filter for the setting name. Delighted `NEXT_PUBLIC_NPS_DELIGHTED_API_KEY` is the canonical reference. Reference: S20-0 AC-9.2 D-12, AC-15 source-inspection.

## Patterns

- Discovered in Epic 20: **Lazy `await import(...)` is mandatory for all vendor SDK imports in Next.js.** Vendor SDKs loaded via dynamic `import('@vendor/sdk')` inside the init function (not top-level static import) — saves the full vendor bundle for non-eligible tier users. Wrap in `try/catch` that LOGS but does NOT throw — vendor-SDK crash MUST NOT break the workspace-page render. The call must occur inside an effect that depends on `user.id` (never before authentication). Reference: S20-0 AC-15.1, fence rows #9/#10; `lib/nps/init.ts`.

## Patterns

- Discovered in Epic 20: **DEFERRABLE INITIALLY DEFERRED constraint pre-check is required for webhook dedup idempotency.** When a UNIQUE constraint is `DEFERRABLE INITIALLY DEFERRED`, `IntegrityError` fires at COMMIT, not at flush — the route handler has already returned. Correct pattern: pre-INSERT `SELECT` short-circuit (`select(Model).where(unique_col == value)`); if found, return 200 with existing row before INSERT. `INSERT ... ON CONFLICT DO NOTHING` doesn't work because it doesn't return the existing row. The DEFERRABLE constraint remains in schema as defence-in-depth. Reference: S20-0 D-17, `nps_feedback.py:315–326`.

## Patterns

- Discovered in Epic 20: **Workspace-scope BEFORE tier-gate in Depends order prevents 402 tier-information leak on cross-tenant attacks.** `require_workspace_role(...)` must be the FIRST dependency, `require_pro_plus_tier` SECOND. A cross-tenant attacker must receive 404 (workspace not visible), NOT 402 (which reveals the feature exists behind Pro+). Fence row #14. Reference: S20-0 AC-5.3, fence #14.

## Patterns

- Discovered in Epic 20: **Module-level dispatch patch targets must be declared in module docstring as testability contract.** All route handler modules that make Celery/Redis calls declare explicit patch targets in the module docstring (e.g., `client_api.api.v1.nps_feedback.send_email`, `client_api.api.v1.nps_feedback.redis_client`). Module-level import with `try/except ImportError` guard ensures patchability in split-package installs. Reference: S20-0 `nps_feedback.py` module docstring; Pass-2 B-1 fix.

## Patterns

- Discovered in Epic 20: **`_set_no_store_headers(response)` called as FIRST line of endpoint — ensures cache-control even on error paths.** Both POST and PATCH handlers call this helper before any business logic. Sets `Cache-Control: no-store` + `Vary: Authorization`. Protects user-specific data (bucket, routed_to) from shared caches even on 4xx/5xx paths. S19-1 H12 carry-forward now a project-wide pattern. Reference: S20-0 `nps_feedback.py:267–276`; Pass-2 C-2 fix.

## Anti-Patterns

- Discovered in Epic 20: **[ANTI-PATTERN] Story file `Status:` not patched to `done` — 18th consecutive epic (E03–E20).** `[ACTION]` IMPACT: `config_tuning` SEVERITY: `critical` — Story 20-0 at `Status: review` in file header despite sprint-status `done` and `REVIEW: Approve` confirmed. The orchestrator `2b-dev-story-verify` phase MUST atomically patch `Status: review → Status: done` in the story file at the same commit as the sprint-status transition. Reference: AP18-C2, AP17-C5, AP19-C1 — all identical action items, all unresolved.

## Anti-Patterns

- Discovered in Epic 20: **[ANTI-PATTERN] DEFERRABLE INITIALLY DEFERRED constraint IntegrityError is unreachable from route handler.** `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `high` — The route handler's `except IntegrityError` clause cannot catch a deferred-constraint violation because the error fires at COMMIT (after ASGI framework returns the response). Dev-story template must state: "For DEFERRABLE INITIALLY DEFERRED unique constraints, use a pre-INSERT SELECT to short-circuit. Never rely on IntegrityError catch from the route handler." Reference: S20-0 D-17, `nps_feedback.py:315–326`.

## Anti-Patterns

- Discovered in Epic 20: **[ANTI-PATTERN] Frontend `User` store type not extended when backend model is extended — disclosure modal silently re-renders forever.** `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `high` — Adding `nps_disclosure_seen_at` to `client.users` without extending the `User` interface in `auth-store.ts` caused the field to be `undefined` (not null), making the disclosure modal never dismiss. Dev-story template gate: "If this story extends a column on `client.users`, verify whether the frontend `User` type in `auth-store.ts` must also be extended; source-inspection test must assert the new field." Reference: S20-0 Pass-1 B-2, `auth-store.ts:21` fix.

## Anti-Patterns

- Discovered in Epic 20: **[ANTI-PATTERN] Spec endpoint path diverged from implementation path — path prefix not included in AC text.** `[ACTION]` IMPACT: `prompt_adjustment` SEVERITY: `medium` — AC specified `POST /api/v1/users/me/nps-disclosure-seen`; implementation mounts at `/api/v1/auth/me/nps-disclosure-seen` (auth router prefix). Divergence only caught at Approve review. Fix: story ACs for new endpoints must include the FULL path with router prefix. "Mount at `auth.py` with prefix `/auth`" = path `/api/v1/auth/me/...`. Reference: S20-0 D-19, Pass-2 Approve N-2 residual.

## Anti-Patterns

- Discovered in Epic 20: **[ANTI-PATTERN] E2E Playwright suite shipped as `test.skip()` at Approve — P0 ACs covered by skipped E2E.** `[ACTION]` IMPACT: `standards_update` SEVERITY: `high` — All 6 Playwright scenarios in `e2e/specs/nps-prompt.spec.ts` remain `test.skip()` at story close (AC-13 is a P0 criterion). E2E green-phase must be a story AC sub-task, not a post-close operational activity. A story with P0 ACs covered only by `test.skip()` E2E must NOT receive Approve — or must explicitly document an environment constraint blocking execution and carry the N-1 residual to [PR] Post-Review. Reference: S20-0 AC-13, N-1 residual, Pass-2 Approve.

## Patterns

- Discovered in Epic 20: entries (→ project-context.md)

## Patterns

- Discovered in Epic 20: | `NEXT_PUBLIC_` as vendor public site-key — documented threat model | high |

## Patterns

- Discovered in Epic 20: | Lazy `await import(...)` mandatory for vendor SDKs in Next.js | high |

## Patterns

- Discovered in Epic 20: | DEFERRABLE INITIALLY DEFERRED constraint → pre-INSERT SELECT required | high |

## Patterns

- Discovered in Epic 20: | Workspace-scope BEFORE tier-gate in Depends order (AP14-04 refinement) | high |

## Patterns

- Discovered in Epic 20: | Module-level patch targets declared in module docstring | medium |

## Patterns

- Discovered in Epic 20: | `_set_no_store_headers` as first line of endpoints | medium |

## Anti-Patterns

- Discovered in Epic 20: + [ACTION] entries (→ project-context.md)
