

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
