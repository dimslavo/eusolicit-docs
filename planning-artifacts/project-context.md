

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
