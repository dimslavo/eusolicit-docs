

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
