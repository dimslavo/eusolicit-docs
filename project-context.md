---
project_name: 'EU Solicit'
date: '2026-05-14'
last_updated_by: 'retrospective-epic-4-amendment-2026-05-14'
last_updates:
  - '2026-05-14: Epic 4 SirmaAI Amendment retrospective — 7 new patterns (E04A-P01..P07) and 7 new anti-patterns (E04A-AP01..AP07) added. Covers stories S04.20–S04.31 (sirmaai-gateway refactor). Key findings: composite circuit-breaker keys (E04A-P01), SQL-guarded converge() repository pattern (E04A-P02), SecretStr end-to-end + log-redaction test (E04A-P03), Standard Webhooks spec + multi-value signature tolerance (E04A-P04), reconciler-as-authoritative-truth eventual consistency (E04A-P05), public-ingress same-commit cleanup (E04A-P06), cross-tenant negative tests embedded in AC templates (E04A-P07). Anti-patterns: single-shot final-patch discipline violated (E04A-AP01 — S04.26 4 review rounds), sprint-status drift on prerequisite stories (E04A-AP02 — S04.20 in-progress while downstream done), reactive NFR assessment (E04A-AP03 — 2nd recurrence), zero ATDD checklists (E04A-AP04 — 2nd recurrence in same epic), 6-epic security/observability carry-forward (E04A-AP05 — terminal escalation pattern), legacy folder co-existence without deletion date (E04A-AP06), multi-round reviews repeating same 3 root causes (E04A-AP07). Retrospective document: eusolicit-docs/implementation-artifacts/epic-4-retro-amendment-2026-05-14.md. NFR report: eusolicit-docs/test-artifacts/nfr-report-epic-04.md (PASS_WITH_CONCERNS, 21/29). Sprint-status NOT modified — epic-4-retrospective:done already set for original scope (2026-04-23); epic-4:in-progress correctly reflects S04.20 pending.'
  - '2026-05-12: Epic 23 retrospective — E22 patterns corrected (E22 retro claimed writes that did not execute; 7 patterns E22-P01..P07 + 4 anti-patterns E22-AP01..AP04 written now). E23 patterns added: 3 patterns (E23-P01..P03) + 4 anti-patterns (E23-AP01..AP04). Key findings: operator-execution stories gate on live-state evidence (E23-P01); SLA calendar soak gate pattern (E23-P02); rescope over cancel (E23-P03); TEA 18th-epic terminal escalation (E23-AP01); sprint-status missing story keys (E23-AP02); zombie code AC (E23-AP03); zero-output guard for file writes (E23-AP04). epic-23-retrospective marked done.'
  - '2026-05-05: Epic 21 retrospective — 7 new patterns (E21-P01..E21-P07) and 4 new anti-patterns (AP21-01..AP21-04) added. k6 load tests as discovery tools, cross-story URL reservation + CI lint gate, HA-by-default via compounding CI gates, code-done vs live-done two-gate spec, blameless incident process via structural enforcement, redis-py resilience hardening, CREATE INDEX CONCURRENTLY + transactional_ddl. Anti-patterns: staging unavailable for NFR stories (AP21-01), FTS without EXPLAIN ANALYZE (AP21-02), TEA as separate backlog (AP21-03 — 16th epic, CRITICAL), Status mismatch (AP21-04 — 18th epic, CRITICAL). epic-21-retrospective marked done.'
  - '2026-04-27: Epic 16 retrospective — 5 new anti-patterns (AP16-01..AP16-07) and 5 new patterns added. New service skeleton delivery failure, port collision cascade, story file status stale after Approve (5th recurrence), circuit-breaker absent from resilience_pattern, epic-spec vs story-spec AC gap, stale traceability matrix, sync_logs schema gap. Patterns: AST structural security tests, shared FernetCrypto, consumer group naming, --fail-on-skipped conftest hook, idempotency per-delivery-target constraint. CLAUDE.md updated with integrations-api port 8007. S16.0 story file Status corrected to done. epic-16-retrospective marked done.'
  - '2026-04-26: Epic 13 retrospective — 7 new patterns (P13-01..P13-07) and 8 new anti-patterns (AP13-01..AP13-08) added. asyncio.CancelledError handler, structlog stdlib routing, fire-and-forget audit canonical form, coordinator story pattern, hot-fix context in Dev Notes, dev-session regression guard, review approval without test execution, sprint-status/story-file discrepancy, AC text update requirement, inj-* priority enforcement, Pydantic Literal for enum fields, asyncio.get_event_loop() deprecation.'
  - '2026-04-25: Epic 12 retrospective supplementary — streaming CSV, TRACE_GATE carry-forward verification, non-functional story entry criteria.'
---

# Project Context for AI Agents

_Critical rules and patterns that AI agents must follow when implementing code in this project. Derived from Epic 1 and Epic 2 retrospective findings, code reviews, and TEA assessments._

---

## Technology Stack & Versions

- **Python:** 3.12+ (all services). Use modern syntax (`type` statements, `match/case`).
- **Framework:** FastAPI with Starlette middleware pattern
- **ORM/DB:** SQLAlchemy (async), Alembic migrations, asyncpg, PostgreSQL 16
- **Cache/Events:** Redis 7 via redis-py (Redis Streams, no Celery)
- **Models:** Pydantic v2 with `BaseSettings` for config
- **Linting:** ruff (I/E/W/F/UP rules), mypy (Python 3.12 target)
- **Testing (backend):** pytest + pytest-asyncio (asyncio_mode="auto"), pytest-cov, Factory Boy
- **Frontend:** Next.js 14 App Router, TypeScript strict, pnpm workspaces + Turborepo
- **Frontend testing:** Vitest (unit/component), Playwright (E2E/browser)
- **Frontend state:** Zustand (persist + devtools middleware), TanStack Query v5
- **Frontend forms:** React Hook Form + Zod (`useZodForm`), shadcn/ui primitives
- **Frontend i18n:** next-intl v3 (`app/[locale]/` route structure, `locales: ['bg', 'en']`, `defaultLocale: 'bg'`)
- **UI components:** shadcn/ui in `packages/ui` (46+ components, barrel-exported)
- **Infra:** Docker Compose (local), Helm 3, Terraform >= 1.5, GitHub Actions CI
- **Logging:** structlog (JSON production, pretty-print dev)

---

## Critical Implementation Rules

### Database

1. **Schema isolation is enforced.** Each service has its own PostgreSQL schema. Service roles have CRUD only on their own schema + SELECT on `shared`. Never write cross-schema queries from application code.
2. **Use migration_role for DDL only.** Application code must not use migration_role connections.
3. **Always specify `schema=` in Alembic migrations.** The mako template lacks a SCHEMA constant — every `op.create_table()` call must explicitly pass the schema parameter. Omitting it creates tables in the wrong schema.
4. **Alembic migration naming:** `NNN_descriptive_name.py` (sequential, not timestamp-based).
5. **version_table_schema** is set per-service in env.py to avoid alembic_version collisions.

### Redis Streams

6. **Use EventPublisher/EventConsumer from eusolicit-common.** Do not use raw Redis commands for event bus operations. The abstraction handles envelope creation, DLQ, and correlation IDs.
7. **Event envelope is mandatory:** `{event_type, payload, timestamp, correlation_id, source_service}`. All events must include all fields.
8. **DLQ is at-least-once.** The xadd→xack sequence is non-atomic. Do not assume exactly-once delivery from DLQ. (Lua script upgrade planned.)

### Shared Packages

9. **All services must import from eusolicit-common** for config, logging, middleware, health, and exceptions. No direct `print()` or `logging.getLogger()`.
10. **Inter-service DTOs live in eusolicit-models.** Event schemas use `Literal` type discriminators for pattern matching.
11. **eusolicit-kraftdata is types-only.** No HTTP client code — only Pydantic models matching the KraftData API.

### Docker & CI

12. **Docker images use multi-stage builds.** Target size < 200MB per service.
13. **Shared packages installed via relative path references** in pyproject.toml (`path = "../../packages/eusolicit-common"`).
14. **CI matrix builds all 8 projects in parallel** (5 services + 3 packages). Each runs ruff, mypy, pytest.

### Testing

15. **Use eusolicit-test-utils** for all test fixtures: DB sessions, Redis clients, JWT generators, Factory Boy factories, API clients.
16. **Test levels:** unit (no external deps), smoke (file/config validation), integration (live Docker services), E2E (Playwright).
17. **Markers:** `@pytest.mark.unit`, `@pytest.mark.integration`, `@pytest.mark.smoke`, `@pytest.mark.slow`, `@pytest.mark.cross_service`.
18. **Test isolation:** Per-test transaction rollback for DB, flush for Redis. No shared mutable state.

### Frontend Architecture

19. **All UI components live in `packages/ui`.** Never create app-local components that duplicate shared functionality. Add to `packages/ui/src/components/` and barrel-export via `packages/ui/src/index.ts`.
20. **Use `<AppShell>` for the authenticated layout.** Both client and admin apps use the shared `<AppShell>` component with sidebar/topbar/children slots. Never rebuild the shell in a feature component.
21. **Data fetching must use `<QueryGuard>`.** All list/table/detail views wrap TanStack Query state with `<QueryGuard isLoading={...} isError={...} isEmpty={...}>`. No ad-hoc `if (isLoading) return <Spinner />` patterns.
22. **Forms use `useZodForm(schema)` + `<FormField>`.** Never use `useForm()` directly. Define the Zod schema in `lib/schemas/`, call `useZodForm(schema)`, and compose `<FormField>` for every input. This wires RHF + zodResolver + inline error display + BG/EN translations automatically.
23. **Auth guard is dual-layer.** Protected routes use `<AuthGuard>` in `app/[locale]/(protected)/layout.tsx` (client-side; shows full-page spinner until `onFinishHydration()`) AND `middleware.ts` (server-side; checks `eusolicit-session` cookie, 307 redirect). Never protect a route with only one layer.
24. **AuthGuard has a 5-second hydration timeout.** If `onFinishHydration()` does not fire within 5 seconds (corrupt localStorage), treat as unauthenticated and redirect to `/login`. Prevents permanent spinner.
25. **Zustand persist keys are namespaced per app.** Use `eusolicit-client-auth-store` and `eusolicit-admin-auth-store` (not a shared key). Prevents cross-app localStorage collision if apps are deployed on the same origin.
26. **`apiClient` uses a Promise singleton refresh-lock for 401s.** All concurrent 401 responses queue until the first token refresh resolves, then retry with the new token. Never call `POST /auth/refresh` more than once per concurrent 401 burst — this triggers E02's token family revocation.
27. **Add `x-request-id: crypto.randomUUID()` to every apiClient request.** This enables correlation between frontend errors and backend logs when observability is operational.
28. **Locale routing uses `localePrefix: 'always'`** with `locales: ['bg', 'en']` and `defaultLocale: 'bg'`. All routes are prefixed: `/bg/dashboard`, `/en/dashboard`. Test that each entry path triggers ≤1 redirect hop using a `countRedirects()` utility in E2E tests.
29. **All UI strings use `useTranslations()` / `getTranslations()` from next-intl.** Never hardcode display strings in components. All namespaces are: `common`, `nav`, `auth`, `forms`, `errors`, `wizard`. BG/EN message files must have identical key sets — enforced by the key-parity check test.
30. **Shell components are server-compatible wrappers.** `<AppShell>`, `<Sidebar>`, `<TopBar>` are server components. Only the interactive leaves (`<SidebarToggle>`, `<UserAvatarMenu>`, `<LanguageSelector>`) carry `'use client'`. Keep this boundary discipline in all feature components.
31. **Wizard state persists in Zustand across page reload.** The `wizardStore` uses `persist` middleware. If `user?.companyId` is undefined at wizard submission, show a validation error — never fall back to a hardcoded stub ID.
32. **Pages using `useSearchParams()` must wrap the component in a `<Suspense>` boundary.** Next.js App Router throws a build error if `useSearchParams()` is called outside a `<Suspense>` boundary. The page default export renders `<Suspense>` wrapping the inner `"use client"` component that reads search params. See `apps/client/app/[locale]/(auth)/callback/page.tsx` as the reference implementation. Applies to any page reading URL state: OAuth callbacks, pagination, filters, redirect params.

### AI Gateway & External Service Integration

47. **All outbound HTTP calls to external services must use the two-layer resilience pattern.** Circuit breaker (outer, per-logical-name) wraps retry (inner, exponential backoff). Never call an external API with only retry or only circuit breaking. The `call_kraftdata()` function in `ai-gateway` is the reference implementation.
48. **Webhook signature validation must use `hmac.compare_digest()`, never `==`.** Applies to all inbound webhooks regardless of provider. Always read the raw request body bytes BEFORE parsing JSON (body bytes are required for HMAC). Unit test must verify both `hmac.compare_digest` is used in source AND that timing difference between valid/invalid signatures is < 1ms.
49. **Agent/service registry YAML must fail fast on startup.** If `config/agents.yaml` is missing, has duplicate entries, or fails Pydantic validation, the service must raise an error and refuse to start. Never silently ignore registry loading failures — this would cause all downstream AI calls to return 404.
50. **Streaming endpoints (`run-stream`) must enforce idle timeout (120s), total timeout (600s), and heartbeat (15s).** Without heartbeat, proxies/load balancers may drop idle SSE connections with no signal to the client. Without timeouts, an abandoned stream holds a semaphore permit indefinitely, blocking subsequent requests.
51. **SSE `StreamingResponse` requires `Cache-Control: no-cache` and `X-Accel-Buffering: no`.** These headers prevent nginx and CDN intermediaries from buffering SSE chunks. Always set on any endpoint returning `text/event-stream`.
52. **`X-Caller-Service` header is required on all AI Gateway execution endpoints.** Used for execution logging, rate limit attribution, and debugging. Return 400 with a descriptive error if missing. Generating services must always pass this header when calling the gateway.

### Authentication & Security

32. **JWT middleware validates only RS256 signature + expiry by default.** Add `audience` and `issuer` claims to `jwt.decode()` calls if multiple services share the same RSA key pair.
33. **`User.is_active` must be checked in every authentication query.** Both `login()` and `refresh_token()` must filter `User.is_active == True`. Deactivated users must not receive new tokens.
34. **Wrap `bcrypt.hashpw` in `run_in_executor`.** `bcrypt` at cost=12 takes 200–400ms and blocks the async event loop. Use `await loop.run_in_executor(None, bcrypt.hashpw, ...)` in every password-setting flow.
35. **All password fields must have `max_length=128`.** Prevents bcrypt HashDoS. Applies to `LoginRequest`, `RegisterRequest`, `PasswordResetConfirmRequest`, `AcceptInviteRequest`, and any future password field.
36. **Rate limiter must fail-open on Redis `ConnectionError`.** Catch the exception, log it as structured warning, and continue processing. Redis outage must not block authentication.
37. **Refresh token rotation requires pessimistic locking.** Use `SELECT ... FOR UPDATE` on the refresh token row to prevent concurrent rotation from forking the token family.
38. **Every company-scoped endpoint must include a cross-tenant negative test.** `Company A credentials → Company B resource → 403`. No exceptions.
39. **OAuth callback must validate the `state` parameter against a session-stored nonce.** Missing or mismatched `state` is a CSRF failure — reject with 400, do not default to empty string. Required before any stub replacement with a real OAuth provider.
40. **`dangerouslySetInnerHTML` usage requires an explicit security review comment.** Any component rendering user-generated content with `dangerouslySetInnerHTML` must include a comment explaining the sanitization applied. Flag in all code reviews from E07 (proposals) onwards.

### RBAC

41. **RBAC uses two-tier enforcement:** company role ceiling (5 levels: admin > bid_manager > contributor > reviewer > read_only) + entity-level permissions (read/write/manage). The role ceiling is always the upper bound; entity permission cannot exceed it.
42. **Admin and bid_manager bypass entity-level permissions for their own company only.** The bypass check must verify `company_id` match before granting access.
43. **`entity_permissions` rows must have UNIQUE constraint on `(user_id, company_id, entity_type, entity_id, permission)`.** Without it, duplicate rows cause `MultipleResultsFound` → 500. Enforce at application layer until DB migration adds the constraint.

### Audit Trail

44. **All mutations (POST/PUT/PATCH/DELETE) must write to `shared.audit_log`.** Capture `entity_type`, `entity_id`, `action_type`, `before` (None for creates), `after` (None for deletes), `user_id`, `ip_address`.
45. **Audit writes must be non-blocking.** Wrap in `try/except`; suppress failures silently (log, do not raise). A failed audit write must never return 500 to the client.
46. **`ip_address` from `request.client.host` is proxy IP behind a reverse proxy.** Future work: read from `X-Forwarded-For` header with proxy whitelist validation.

---

## Patterns (Do This)

### From Epic 1 Retrospective

- **ATDD-first for infrastructure/security stories.** Write failing acceptance tests before implementation. This produced 420 tests for schema isolation (Story 1.3) and caught real issues.
- **Risk-driven test investment.** Allocate more tests to high-risk items (Score ≥ 6). Stories 1.3 and 1.4 got 140 and 246 tests/point respectively.
- **Deferred work tracking with rationale.** Every code review item that isn't fixed immediately goes to `deferred-work.md` with source story, description, and reason for deferral.
- **Regression verification per automation round.** Each TEA automation run must verify the full test suite, not just new tests.
- **Shared test utilities early.** Create reusable fixtures in eusolicit-test-utils before writing feature tests.

### From Epic 2 Retrospective

- **Transaction-rollback fixture pattern for API integration tests.** Every API test fixture must use `await session.rollback()` in a `finally` block alongside `fastapi_app.dependency_overrides.clear()`. This is the gold standard for per-test DB isolation confirmed by TEA (Story 2.12: Isolation 100/100).

  ```python
  # Canonical pattern — replicate for every new API test fixture
  async with client_api_session_factory() as session:
      fastapi_app.dependency_overrides[get_db_session] = override_db
      try:
          yield client, session, access_token, company_id_str
      finally:
          fastapi_app.dependency_overrides.clear()
          await session.rollback()
  ```

- **Deterministic role-injection helper (`_register_and_verify_with_role`).** Null out TempCo membership's `accepted_at` before inserting the target membership to ensure `auth_service.login()` deterministically encodes the target company and role in the JWT. Reuse verbatim in any test verifying role-based authorization.

- **Exhaustive parametrized permission matrix for RBAC stories.** Cover all N roles × M permissions × own-company/cross-company combinations with `@pytest.mark.parametrize`. No partial coverage of the permission matrix is acceptable.

- **Cross-tenant negative tests are mandatory for every company-scoped endpoint.** Register Company A and Company B in the same rollback-scoped session. Verify Company A credentials return 403 on all Company B resources.

- **Parametrize >2 structurally identical tests.** If two or more test methods differ only in one variable (e.g., missing field name, role name, error code), use `@pytest.mark.parametrize`. The project already uses this pattern — apply it consistently.

- **Extract cross-story test helpers to eusolicit-test-utils at the first reuse.** Don't wait for the third story to discover the pattern; extract shared helpers as soon as the second story needs them.

- **Contribute all mutations to the shared `shared.audit_log`.** New stories introducing POST/PUT/PATCH/DELETE endpoints must write audit entries. Audit writes are non-blocking (try/except).

### From Epic 3 Retrospective

- **`<QueryGuard>` is mandatory for list/table/detail views.** Every component that fetches data with TanStack Query must wrap its state transitions in `<QueryGuard>`. Ad-hoc `if (isLoading) return <Skeleton />` scattered through feature components is the anti-pattern this replaces.

- **`useZodForm(schema)` + `<FormField>` is the only form construction pattern.** Define the Zod schema first, call `useZodForm(schema)`, compose `<FormField>` variants. This gives you zodResolver, inline animated error display, BG/EN i18n translation, and RHF controller wiring for free.

- **All new feature UI goes into `packages/ui`.** Do not create app-local `components/` folders for shared elements. If a component is used by both client and admin apps, or might be in the future, it belongs in `packages/ui`.

- **Locale routing changes must include a redirect count assertion.** Any PR touching `middleware.ts` or `i18n.ts` must include a Playwright test asserting ≤1 redirect hop for the affected routes, using the `countRedirects()` pattern from `locale-redirect.spec.ts`.

- **Produce a Stub Removal Checklist alongside any stub implementation.** When a story introduces MSW mocks, hardcoded IDs (e.g., `stub-company-id`), or stub API responses, the story deliverables must include a checklist documenting every stub and which epic/story will replace it. This prevents silent integration failures.

- **Dual-layer auth guard pattern is established — apply it without deviation.** `<AuthGuard>` (client-side spinner until hydration) + `middleware.ts` (server-side cookie check) is the standard. New routes in the `(protected)` group automatically inherit the client guard. Ensure `middleware.ts` matchers cover any new route paths.

- **Toast notifications via `useToast()` from `packages/ui`.** Call `toast.success(title)`, `toast.error(title)`, `toast.info(title)` — not custom alert patterns. The toast system handles stacking, auto-dismiss timers, and portal rendering.

### From Epic 4 Retrospective

- **All outbound HTTP calls use a two-layer resilience pattern: circuit breaker (outer) wrapping retry (inner).** `circuit.call(lambda: with_retry(http_factory, agent_name=...))` — circuit rejects immediately when OPEN; retry applies exponential backoff (1s → 2s → 4s, ±25% jitter) for 5xx/timeout/connection errors only. 4xx and `CircuitOpenError` are non-retryable and propagate immediately through both layers. Apply this pattern to any new outbound HTTP client in subsequent epics.

- **YAML-driven registry pattern decouples business logical names from external provider IDs.** `config/<service>.yaml` maps logical names to UUIDs/identifiers/types; loaded at startup via Pydantic models; fails fast on missing/duplicate entries; supports hot-reload via admin endpoint guarded by `asyncio.Lock`. Use for any integration where external IDs may be rotated independently of business logic.

- **All inbound webhooks signatures must use `hmac.compare_digest()` — not string equality (`==`).** Naive string comparison is vulnerable to timing attacks. Import from stdlib `hmac` module. Unit test must inspect source code for `compare_digest` usage AND verify constant-time behavior with an off-by-one-byte signature under a timing assertion (< 1ms difference). Required for any webhook endpoint regardless of perceived risk.

- **Audit and logging writes must use `asyncio.create_task()` (fire-and-forget).** DB writes for audit trails, execution logs, and webhook logs must never block the response path. Wrap in `asyncio.create_task(_write_to_db(...))`. The handler must catch all DB exceptions and emit a structured ERROR log with enough context (execution_id, entity_id) for manual recovery. Logging failure must NOT propagate a 500 to the caller.

- **testcontainers + respx is the mandatory backend integration test stack.** testcontainers provides real PostgreSQL and Redis instances; `respx.MockRouter` mocks all outbound HTTP (KraftData, external APIs); `httpx.AsyncClient(transport=ASGITransport(app, lifespan="on"))` runs the full FastAPI stack in-process. This combination is CI-safe (no live credentials), deterministic (no network flakiness), and exercises the complete request lifecycle including DB writes and Redis publishes. All backend epics must produce an integration test suite using this stack.

- **Admin introspection endpoints are mandatory for all stateful backend services.** Every service that owns stateful resilience mechanisms (circuit breaker, rate limiter, connection pool, async queue) must expose ClusterIP-only admin endpoints: `GET /admin/circuits`, `GET /admin/rate-limit`, `GET /admin/executions`. These are operational observability surfaces — required as acceptance criteria in any story implementing stateful components.

- **SSE streaming responses require `Cache-Control: no-cache`, `X-Accel-Buffering: no` headers.** Without these, nginx/CDN proxy layers buffer SSE chunks and the client never receives events in real time. Always set these response headers on `StreamingResponse` for SSE endpoints. Also forward `X-Request-ID` in response headers.

- **Per-instance (in-memory) circuit breaker state is single-replica only.** Acceptable for initial deployment. When auto-scaling is planned, circuit state must be migrated to Redis so all replicas share the same failure count and cooldown state. Document this limitation explicitly at implementation time; create a pre-scale backlog item.

- **`asyncio.Semaphore` for concurrency control must track active/queued/rejected counts for observability.** Rate limiter state (active requests, queued requests, total rejected) must be exposed via an admin endpoint. Streaming requests hold their semaphore permit for the full stream duration — design for this in capacity planning (streaming load starves sync calls).

- **Integration test mocks must assert the exact Authorization header format, not just that a request was made.** When using respx (or any HTTP mock library) to mock outbound calls to external providers, always assert the exact auth scheme with `.match(headers={"Authorization": re.compile(r"Bearer .+")})` or equivalent. A wrong auth scheme (e.g., `X-API-Key` instead of `Authorization: Bearer`) survives the entire integration test suite and is only caught in production. This applies to all outbound HTTP mocks — KraftData, Stripe, SendGrid, Google OAuth — and to any PR that refactors HTTP client configuration. (Source: E04 S04.02 critical review-fix, 2026-04-23.)

### From Epic 9 Retrospective

- **Dual-layer consumer idempotency: DB constraint + Redis SETNX.** For all Celery Redis Stream consumers, use `INSERT ... ON CONFLICT DO NOTHING` on a uniqueness constraint at the DB layer AND a `SETNX` per-recipient key at the Redis layer. This prevents duplicate email dispatch under at-least-once delivery even during crash-before-ACK scenarios. Test both layers independently and together.

- **`asyncio.to_thread` wrapper is mandatory for Celery `.delay()` calls inside async consumers.** When a Celery task is dispatched from within an `async` def consumer (e.g., a Redis Stream consumer running in the asyncio event loop), wrap the synchronous `.delay()` call: `await asyncio.to_thread(task.delay, ...)`. Without this, the synchronous Celery call blocks the event loop. Add a test asserting `asyncio.to_thread` is called rather than `.delay()` directly.

- **Return 404 (not 403) for cross-user scoped resource access.** When a user accesses another user's preference, calendar connection, or any user-scoped entity, return 404. 403 leaks existence. Test both: own resource → 200, other user's resource → 404. Apply to all CRUD endpoints.

- **Narrow `except` clauses in Celery tasks — never swallow `celery.exceptions.Retry`.** Broad `except Exception:` handlers will swallow `celery.exceptions.Retry`, silently converting retryable errors into permanent failures. Always use narrow exception types or explicitly re-raise `Retry`. Add a test `test_retry_exception_propagates` for any task that has a retry path inside a try/except.

- **ECDSA / HMAC webhook signature validation is mandatory for all inbound webhooks.** Validate raw `request.body()` bytes before JSON parsing. Use provider verification library where available (SendGrid: `EventWebhook.verify_signature`). Fail-closed on empty/missing key. Three required tests: valid → 200, invalid → 401/403, empty-key → 401/403. This is the same `hmac.compare_digest()` rule from E04 extended to ECDSA signatures.

- **Fernet encryption canonical module for third-party OAuth tokens.** All OAuth2 refresh/access token storage must use the `notification/core/token_crypto.py` pattern: Fernet roundtrip test (stored bytes ≠ plaintext, decrypt(encrypt(t)) == t), key sourced from env/Vault, value scrubbed from structlog. Reuse this module — do not re-implement per-service.

- **Stripe usage counter ordering: read → API(idempotency-key) → GETDEL.** This exact sequence prevents both double-billing (idempotency key) and data loss (GETDEL only after confirmed Stripe 200). Two required P0 tests: counter intact on failure, counter cleared on success.

- **Beat schedule constants in the epic spec must be cross-checked against the Beat dict before story creation.** Any story that references a UTC time constant in its ACs must be reconciled against the service's Beat schedule dict (e.g., `BEAT_SCHEDULE` in `beat_schedule.py`). Divergences between spec and implementation produce tested-but-wrong schedules that may not be caught until operational monitoring.

### From Epic 10 Retrospective

- **`bmad-testarch-trace` traceability workflow must run BEFORE the first story begins implementation.** The matrix output is the AC coverage plan — every story knows which AC it satisfies before coding starts. This is the causal factor for Epic 10's first-ever TRACE_GATE PASS. Running it as a retrospective artifact is too late; it must gate the dev queue.

- **Foundational security middleware (e.g., S10.02 proposal-level RBAC) is a blocking gate.** Do not merge features that rely on a security layer until that layer is implemented and verified. UUID-based existence leakage is a high-severity risk. Stories depending on an unimplemented security dependency must not enter `review` status until the security story is `done`.

- **Atomic UPSERT with `ON CONFLICT DO UPDATE WHERE (expires_at < NOW() OR locked_by = EXCLUDED.locked_by)` is the correct serialization point for pessimistic locking.** No application-layer lock loop needed. A single database round-trip either acquires the lock atomically or returns the existing holder's info. The concurrent acquisition test must assert that a second requester with a valid lock gets 423 with lock-holder name and expiry. (Source: S10.03 section locking.)

- **Upsert field partitioning for AI + human decision flows.** When an endpoint stores both AI recommendation and human decision, partition the two operations so `evaluate` preserves `decision`/`decided_*` fields and `decide` preserves `ai_recommendation`/`evaluated_at`. Neither call clobbers the other's fields. Service returns `(response, was_insert)` tuple; router maps to 201/200 accordingly. (Source: S10.10 bid/no-bid decision.)

- **Visibility-aware polling for async AI results.** Any frontend component polling an async AI operation (lessons learned, report generation, AI scorecard) must: (1) bind `document.visibilitychange` to pause polling when tab is hidden and resume on return, (2) use a per-mount attempt counter (not a global variable), (3) cap with a wall-clock bound (e.g., 15s × 20 = 5 min max). Prevents runaway polling accumulating across browser tabs. (Source: S10.16 outcome-lessons poll.)

- **`lessons_learned_status` tracking column pattern for async AI agent results.** Stores `pending` / `in_progress` / `complete` / `failed`. Background task runs with an independent DB session. Poll endpoint reads status and result JSONB. This pattern is reusable for any async AI integration where the caller should not block waiting. (Source: S10.11 bid outcome.)

- **SQL injection structural test required for any story using SQLAlchemy `text()` or raw SQL.** String interpolation on dynamic values in `text()` clauses is an injection risk invisible to ruff, mypy, and ATDD. Add `test_{feature}_uses_bound_params` to the ATDD checklist: assert the function uses `bindparam()` or named parameters, never f-strings on user-supplied values. (Source: S10.09 approval decision engine — caught and fixed in review.)

- **Celery task registration structural test is mandatory.** A function missing `@app.task` imports and unit-tests normally but is never registered in the worker; the production call silently does nothing. Every story introducing a Celery task must include: `assert "service.module.task_name" in celery_app.tasks`. One line; prevents silent no-op deployment. (Source: S10.03 section lock cleanup task.)

- **Upsert endpoints must return 201 on insert and 200 on update — not always 201.** RFC 7231: 201 = resource created; 200 = resource updated. ATDD must assert both: first call → 201, second call with same payload → 200. Service layer returns `(response, was_insert)` to route accordingly. (Source: S10.10 — always returned 201 regardless, caught in review.)

- **Frontend stories covering 3+ distinct user workflows must be split before entering the dev queue.** S10.16 covered approval stepper + bid/no-bid radar chart + outcome recording in one story and generated 37 review action items (10 blocking). The correlation is systematic — this scope pattern always produces review debt. Story LOC > 3,000 is a split signal.

- **Pre-story spec consistency check for FK + CHECK constraint interactions.** When a story's ACs include a CHECK constraint on a nullable column that also has ON DELETE SET NULL on its FK, the constraint is contradictory (FK nullifies the column; CHECK rejects null). Resolve before implementation begins. (Source: S10.04 comments — `resolved_by IS NOT NULL` in CHECK vs FK `ON DELETE SET NULL`.)

- **Exhaustive audit assertions in every backend story.** Verify `action_type`, `entity_type`, `entity_id`, and `after` JSON content for all POST/PATCH/DELETE endpoints. Enforce PII guards (masking bodies in audit logs).

### From Epic 5 Retrospective

- **`SELECT FOR UPDATE SKIP LOCKED` is mandatory for Celery batch queue workers.** Any Celery Beat task that polls a DB table for pending work and processes items in batches must use `SELECT ... WHERE status='pending' FOR UPDATE SKIP LOCKED`. Without it, multiple worker replicas race to claim the same items, producing duplicate processing and state corruption. Items must be marked `processing` before the AI Gateway call and restored to `pending` on failure or `failed` after max retries. See `process_enrichment_queue.py` as the canonical reference.

- **Celery `task_failure` signal handlers must use Redis-backed state, not in-memory dicts.** Module-level Python dicts (`_RUN_ID_REGISTRY: dict[str, str]`) are incompatible with Celery prefork pool: the signal fires in a different OS process — `_pop_run()` returns `None` and audit records are never updated. Use `redis.setex(f"service:run_id:{task_id}", ttl=3600, value=str(run_id))` and `redis.getdel(f"service:run_id:{task_id}")` to correlate signal handlers with audit rows across process boundaries.

- **3-queue Celery isolation prevents scoring/guide work from starving crawl tasks.** For Celery applications mixing task types with different resource costs (quick crawl HTTP calls vs. long per-item AI scoring), use dedicated queues per task type: `pipeline_crawl`, `pipeline_scoring`, `pipeline_guides`. Start workers with `--queues <queue_name>`. Set `worker_prefetch_multiplier=1` + `task_acks_late=True` for fair dispatch. Without isolation, a scoring wave (100+ tasks) starves crawl tasks queued behind them.

- **Dedicated `CollectorRegistry` is mandatory for Prometheus metrics in FastAPI services.** Define `SERVICE_METRICS_REGISTRY = CollectorRegistry()` in a `metrics.py` module; register all metrics (`Histogram`, `Counter`, `Gauge`) to it (not the global default). The `/metrics` endpoint calls `generate_latest(SERVICE_METRICS_REGISTRY)`. This prevents `ValueError: Duplicated timeseries in CollectorRegistry` when pytest reimports modules. See `services/data-pipeline/src/data_pipeline/metrics.py` as canonical.

- **Pagination sentinel termination must use a falsy check.** When consuming a paginated external API where the "end of pages" token may be `None`, `""`, `0`, or a missing key, use `raw_token = response.output.get("next_page_token"); page_token = str(raw_token) if raw_token else None`. A truthy check covers all edge cases; `if raw_token is not None:` does not cover empty-string sentinels.

- **Celery eager mode + testcontainers + respx is the E2E integration test pattern for pipeline services.** For full-chain testing (Beat → task chain → DB writes → Redis stream publish), use `task_always_eager=True`, session-scoped PostgreSQL testcontainer, and `respx.MockRouter`. Runs the complete chain in a single process without a real broker; CI-safe; verifies side effects end-to-end. See `test_e2e_pipeline.py` as canonical.

### From Epic 12 Retrospective

- **All stories with frontend ACs must have a Playwright E2E spec file before being marked `done`.** A RED-phase spec (test.skip) is the minimum deliverable. The spec must be activated (GREEN) before the epic retrospective. Absence of a spec file is the same as absence of implementation evidence for frontend ACs.

- **Tier gate enforcement requires E2E API tests covering all 4 tiers.** For every Professional+ or Enterprise-gated feature, test all tier combinations: Free→403, Starter→403, Professional→200 (or 403 if Enterprise-only), Enterprise→200. Reuse the `tier-gate-enforcement.api.spec.ts` pattern.

- **Analytics charts (Recharts) must be E2E testable via SVG presence assertion.** When implementing Recharts `<BarChart>`, `<LineChart>` or `<RadialChart>` inside `<QueryGuard>`, the E2E test should assert: (1) `<svg>` element rendered, (2) at least one data bar/line/point present, (3) hover tooltip appears on mouseover. Empty state must show `<EmptyState>` component from `packages/ui`, not a blank chart area.

- **Async Celery tasks require integration-level end-to-end testing.** It is not sufficient to test the API endpoint that enqueues the task. The test must verify: (1) the Celery task was executed, (2) the side effect occurred (S3 object created, email sent via mock, DB row updated), (3) the job status endpoint reflects completion. Use LocalStack for S3 and a SendGrid mock for email in CI.

- **S3 signed URL delivery must be asserted — not assumed.** Any story that stores output in S3 and returns a signed URL (report generation, proposal export, document uploads) must include a test that: (1) verifies the S3 object exists at the expected key, (2) resolves the signed URL via HTTP and receives a 200 with expected content-type, (3) asserts the URL expires within the specified window (24h for reports).

- **VPN-restricted admin services use a single IPAllowlistMiddleware test suite.** Do not duplicate IP restriction tests per-story. Create one comprehensive `test_ip_allowlist.py` covering all routes in the service, executed once. Reference it in each story's traceability entry.

- **RED-phase ATDD specs require an owner and an activation story at creation time.** When `bmad-testarch-atdd` generates test.skip() specs, the story deliverables must include: (1) the spec file, (2) a reference to the story that will activate it (remove test.skip and achieve GREEN). No RED-phase spec should exist at epic close without a named activation story.

- **Non-functional stories (load testing, security audit) must not be marked `done` until evidence files contain actual results.** A template with placeholder dashes is not done. The `load-test-results.md` must contain real p50/p95/p99 numbers. The `security-audit-checklist.md` must have all checkboxes filled with evidence and a sign-off. Entry criteria include confirming the staging environment and tooling are available before the story begins.

- **TEA automation review is a per-story gate, not an epic-level activity.** Update `tea_status` in sprint-status.yaml for each story as it is reviewed. `tea_status: {}` at retrospective time means the quality validation layer was skipped for the entire epic.

### From Epic 6 Retrospective

- **TierGate MUST be a FastAPI per-route `Depends()` on every endpoint returning gated data — never middleware.** Implementing tier enforcement as a per-route dependency makes it impossible to accidentally omit from new endpoints. The strict response-model selection (e.g., `OpportunityFreeResponse` 6-field-only) must happen inside the gate dependency, not in the route handler. A `test_free_response_has_exactly_six_model_fields` unit test prevents field leakage through model evolution. This is the definitive pattern for all revenue-critical tier enforcement.

- **Atomic Lua script (`_LUA` module-level constant) is mandatory for all Redis metering operations.** Never separate GET + INCR across two Redis round-trips — the race window corrupts billing signal. Use `redis.eval()` with a single script: reads current value, returns 429 if ≥ limit, increments and sets EXPIRE atomically. `fakeredis[lua]>=2.21` (lupa package) required for EVAL support in unit tests; testcontainers Redis required for concurrency race tests. See `usage_gate.py::_USAGE_LUA` as canonical.

- **Negative ATDD assertions are mandatory for stories involving streaming protocols (SSE, WebSocket).** For any SSE endpoint, the ATDD checklist must explicitly assert: "Component does NOT use `new EventSource`" (EventSource is GET-only; POST SSE endpoints silently fail). Also assert "Component does NOT use `Axios` for streaming body" (Axios buffers the full response). Use native `fetch` + `ReadableStream` for POST SSE. These structural negative assertions prevent invisible runtime failures that no functional test catches.

- **URL-driven state is mandatory for all filter/sort/pagination UI.** Filter sidebar, sort controls, and pagination must use `useSearchParams()` + `router.replace()` — never `useState`. Add stale-closure prevention via `searchParamsRef = useRef(searchParams)` updated on every render. This enables shareable URLs, survives client-side navigation, and eliminates the lost-state-on-back-button problem.

- **Headless orchestration components (`'use client'` + `return null`) must receive store accessors as parameters.** Components that register Axios interceptors or global effects should render `return null` and mount exactly once in layout. The accessor function must be passed as a parameter (e.g., `show` from `useUpgradePromptStore`) rather than called with `getState()` inside — prevents circular dependency between `packages/ui` and `apps/client`. Always call `return Promise.reject(error)` in error handlers to preserve TanStack Query error states.

- **Dual-session service functions are required for any service spanning the pipeline/client schema boundary.** Functions reading from `pipeline.*` use `get_pipeline_readonly_session` (separate `MetaData(schema="pipeline")` instance); functions writing to `client.*` use `get_db_session`. Never reuse the same session for both schemas. This prevents cross-schema FK violations and enforces the architectural boundary.

- **UsageGate check MUST execute before `StreamingResponse` is created.** Once `StreamingResponse` is returned to FastAPI, HTTP headers (200 + `Content-Type: text/event-stream`) are committed. A 429 raised inside the generator after headers are sent cannot change the status code — the client receives a success stream followed by an error event. Always check and increment quota atomically, then return `StreamingResponse(...)`.


### From Epic 7 Retrospective

- **Security hardening story injection is the correct NFR FAIL response.** When NFR assessment returns FAIL on security/maintainability, inject a P0 blocking hardening story (reference: S07.17). ACs must derive directly from NFR finding items; the story resolves within the same sprint. When security mitigations are designed-in from story ACs rather than retrofitted, NFR assessment produces PASS (with CONCERNS) instead of the FAIL→injection cycle.

- **Content-hash optimistic locking is the standard for collaborative document editing.** Use a `content_hash` column (SHA-256), client sends the current hash in the request body, `SELECT FOR UPDATE` + hash comparison inside a transaction, return `409 Conflict` with a structured body on mismatch, and display a conflict resolution dialog on the frontend. Both backend (content save API) and frontend (conflict dialog) stories are required to close R-001-class data loss risks.

- **`ThreadPoolExecutor` is mandatory for CPU-bound library calls in FastAPI.** Any endpoint calling WeasyPrint, python-docx, Pillow, lxml, or any CPU-bound library MUST use `await loop.run_in_executor(executor, fn, *args)`. Event loop blocking is invisible in unit tests and only manifests under load. ATDD checklists for export/render stories must assert "function uses `run_in_executor`".

- **Reset-stuck background task is required for all long-running async state machines.** Any resource that transitions through an in-progress status (generation, scanning, export, crawler run) MUST have a Celery Beat cleanup task marking stale entries `FAILED` after a configurable TTL. Reference: S07.05's `reset_stuck_proposals_task`. Add as a required AC to all stories introducing a `status` field with an in-progress variant.

- **TEA test review must gate `review → done` story transitions; minimum score ≥ 80/100.** TEA review catches backend/frontend contract schema drift and spec-vs-implementation constant contradictions invisible to ATDD. S07.11 TEA review (95/100) caught `ProposalResponse` missing `current_version_number`/`generation_status` and a 1280px vs 1024px breakpoint contradiction that unit tests did not surface.

- **Vitest source-inspection ATDD scales to complex multi-panel frontend components.** Regex and import assertions without runtime rendering are stable in CI and enable RED-phase tests before components exist. Reserve RTL/JSDOM for user-interaction flows; S07.15 (87 tests, all GREEN) is the canonical reference.

### From Epic 8 Retrospective

- **Clean billing service boundary is the reference architecture for payment-adjacent services.** Separate modules: `billing_service.py` (Stripe customer/subscription), `webhook_service.py` (idempotent event processing), `vies_service.py` (VAT/VIES with fallback), `tier_gate.py` (FastAPI `Depends()` gating), `tier_cache.py` (Redis + DB fallback), `api/v1/billing.py` (thin router). No cross-service DB joins; all events via Redis Streams; 100% isolated in `client-api/services/billing/`.

- **External validation services on the registration critical path must fail-open to `pending`.** VIES (or any external validation service) returning 503/timeout MUST NOT block registration. Pattern: timeout/503 → `status: pending`; return 201 to caller; retry on next billing event; apply reverse-charge only on `status: valid`. Three integration tests required: valid → synced, invalid → 422, service down → pending+200.

- **External account provisioning (Stripe, CRM) must run as a `BackgroundTask` after the resource creation endpoint returns 201.** The provisioned ID is nullable until populated. All billing endpoints return 422 with a structured error on missing ID. This prevents external service outages from blocking user registration.

- **Webhook dedup table with `event_id` unique constraint is the mandatory idempotency mechanism for all payment webhooks.** Pattern: `INSERT INTO webhook_events(stripe_event_id)` inside the processing transaction; unique-constraint violation → return 200 (already processed). Not in-memory state or Redis TTL. Reference: S08.04 `webhook_events` table.

- **Event-driven cache invalidation must DELETE the cached key, never SET it to an assumed new value.** `DELETE` forces the next request to read the authoritative DB value. `SET` requires knowing the new value at consumer time — creating a read-before-write race. Reference: S08.14 `tier_cache.delete(company_id)` pattern.

- **`asyncio.to_thread()` is mandatory for all synchronous I/O SDK calls inside `async def` FastAPI handlers.** Stripe Python SDK, VIES SOAP clients, and any synchronous HTTP/DB library MUST be wrapped with `await asyncio.to_thread(sync_fn, *args)`. This extends the E07 ThreadPoolExecutor rule (CPU-bound) to I/O-bound synchronous SDKs.

### From Epic 13 Retrospective

- **`asyncio.CancelledError` requires an explicit handler in Python 3.8+.** `CancelledError` inherits from `BaseException` — a bare `except Exception:` silently drops cancellation signals, leaving `async with` / `finally` blocks in partially-committed state. Always add `except asyncio.CancelledError: raise` (or explicit cleanup) before the generic `except Exception` branch in any coroutine with a `finally` block. Add to ATDD checklist for stories introducing `async` endpoints with cancellation-sensitive paths. (Source: S07.17 Task 4 bug fix.)

- **`structlog` stdlib routing is required for `pytest caplog` to capture log records.** `structlog.PrintLoggerFactory()` (the dev default) bypasses stdlib `logging.Logger`; `pytest`'s `caplog` fixture only captures stdlib records. Add a session-scoped autouse fixture to every service's `tests/conftest.py`:
  ```python
  @pytest.fixture(scope="session", autouse=True)
  def configure_structlog_stdlib():
      structlog.configure(logger_factory=structlog.stdlib.LoggerFactory())
  ```
  Without this fixture, log-capture assertions pass vacuously (nothing captured). (Source: S07.17 Task 5.9.)

- **Fire-and-forget audit write canonical form (E13-authoritative extension of E04 Rule 45).** The background task creates its own `async with session.begin()`, catches all exceptions, logs `audit_write_failed` at ERROR without propagating, and is scheduled from the endpoint's `finally` block so cancellation paths are also audited. The `after` field is populated from the terminal outcome known at `finally` time. Prior forms using `await write_audit_entry()` + `await session.commit()` inline are prohibited on the TTFB path. (Source: S07.17 F4/F2 resolution.)

- **Coordinator story pattern for multi-concern hardening epics.** When a hardening epic has ≥3 independent work streams, introduce a coordinator story that: (1) delegates narrow technical slices to sub-stories each with their own AC set, (2) owns cross-cutting gate criteria verifiable only after sub-stories close, (3) explicitly maps which sub-story satisfies which coordinator AC. Prevents single mega-stories with 10+ ACs and long review cycles. (Source: E13 drift-recovery-story.)

- **Hot-fix context section in story Dev Notes.** Any story hardening an already-merged emergency inline fix must include a "Hot-Fix Context" section specifying: commit SHA + message, exact files and line numbers changed, and what the story must add beyond the fix. Without this section, dev agents re-implement already-merged changes. (Source: S07.17 "Hot-Fix Context" with commit `2d41fcf`.)

- **Migration spec contradiction ("no migration needed") is predictable and preventable.** Before writing "no migration needed" in any story, grep the target ORM models and alembic versions to confirm the schema matches spec assumptions. Add to story template: "Migration required? [YES/NO — confirmed by grepping target schema]". (Source: S07.17 migration 025 — `shared.audit_log` missing two required columns despite spec claim.)

### From Epic 19 Retrospective

- **WeasyPrint thread-pool timeout recovery requires `_reset_executor()`, not `future.cancel()`.** When `asyncio.wait_for()` raises `TimeoutError` on a `run_in_executor` call, the underlying `concurrent.futures.Future` may still be executing — `future.cancel()` has no effect on a running thread. The correct pattern: on `TimeoutError`, call `self._executor.shutdown(wait=False)` and reassign `self._executor = ThreadPoolExecutor(max_workers=1)` before logging and re-raising. Without executor reset, subsequent render calls inherit the stalled thread and time out indefinitely. Add to ATDD checklists for all PDF/export stories: "TimeoutError path calls `_reset_executor()`, not `future.cancel()`". (Source: E19 S19-2 B-3.)

- **`CurrentUser` JWT principal does NOT carry `workspace_id`.** The `CurrentUser` dataclass built from the RS256 JWT payload contains only `user_id`, `company_id`, `role`, and `subscription_tier`. Any endpoint that needs `workspace_id` must derive it from: (a) the URL path parameter (`/{workspace_id}/...`), (b) a request body field, or (c) a FK join on the target resource. Accessing `current_user.workspace_id` raises `AttributeError` at runtime with no fallback. Add to ATDD checklist for every story introducing workspace-scoped endpoints: "No code path reads `current_user.workspace_id`". (Source: E19 S19-2 B-1.)

- **Shared PDF/HTML Jinja2 templates must live in `packages/eusolicit-common/report_templates/`.** Any template used by more than one service belongs in `eusolicit-common/report_templates/<feature>/` and is referenced via `importlib.resources.files("eusolicit_common.report_templates.<feature>").joinpath("<template>.html")`. Duplicating a template per-service (e.g., a second copy of `outcome_brief.html` inside `notification-api/`) is a blocking violation: template drift produces divergent output between scheduled and on-demand generation paths. (Source: E19 S19-2 B-12.)

- **`chord(group(...), callback.s())` is required for Celery fan-out tasks that need a consolidated result.** When a task fans out N sub-tasks and requires a single downstream action (email, PDF write, DB aggregate), use `chord(group(subtask.s(arg) for arg in items), aggregate_callback.s())`. Firing N `apply_async()` calls and scheduling a separate aggregator provides no causal barrier — the aggregator has no guaranteed ordering relative to sub-task completion. The chord back-end (Redis) handles the fan-in barrier. (Source: E19 S19-2 B-6.)

- **S3 content-hash dedup must compare `Metadata['content-hash']`, not key existence.** Before uploading, call `head_object(Bucket, Key)` and compare `response['Metadata']['content-hash']` against the SHA-256 of the new payload. Existence-only dedup (key present → skip) silently serves stale content when the payload changes but the S3 key does not (e.g., same company/month/workspace, regenerated brief). Metadata comparison is O(1) — no body download required. (Source: E19 S19-2 B-13.)

### From Epic 21 Retrospective

- **Load tests are discovery tools — run `EXPLAIN ANALYZE` before any FTS story.** (E21-P01) Any story measuring FTS performance must include a story entry-criteria item: "Run `EXPLAIN ANALYZE <fts_query>` on the target table; confirm Bitmap Index Scan (not Seq Scan)." Seq Scan on a TSVECTOR column without a GIN index always produces a performance HALT at 1M+ rows (PE.01 captured 289ms at 10K rows — projecting to ~28s at 1M). This check takes 30 seconds; omitting it cost PE.01 a revision pass and PE.02 a required migration (`M_PE02_opportunities_tsv_gin_index` with `CREATE INDEX CONCURRENTLY`). If Seq Scan is found: create the GIN migration before measuring performance. (Source: PE.01 AC-2.4 Pass-7 B10.)

- **Cross-story URL reservation via placeholder annotations + CI referential integrity gate.** (E21-P02) When a story produces artifacts referenced by URL in a downstream story, commit placeholder paths in code that the downstream story fulfills. Combine with a CI lint gate (e.g., `check_runbook_url_coverage.py`) asserting file existence + required section headers. This prevents the "placeholder that shipped to production" defect class. Pattern: PE.05 commits placeholder `runbook_url` annotations in alerting rules; PE.06 fulfills all 12 runbook files; PE.06's lint gate verifies the closed loop on every PR.

- **HA-by-default invariant enforcement via compounding CI lint gates.** (E21-P03) When an epic establishes an invariant, add a CI lint gate in the same story making it impossible to violate silently. E21 introduced three compounding gates: `check_helm_pdb_and_minreplicas.py` (PE.04: PDB + `minReplicas ≥ 2` required for all services), `test_metrics_endpoint_contract.py` (PE.05: `/metrics` HTTP 200 + Prometheus exposition format required on all FastAPI services), `check_runbook_url_coverage.py` (PE.06: `runbook_url` annotation must resolve to a file with 6 required sections + `SLA-Scope` header). Future infrastructure stories cannot regress these invariants without CI failing.

- **"Code done" vs "live done" must be two explicit gates in infrastructure epic specs.** (E21-P04) For epics with operator-deferred production steps (live cutovers, chaos drills, soak gates), define two gates: (1) CODE GATE — all Terraform, Helm, and config code merged + CI-passing; (2) LIVE GATE — all operator production steps executed + soak period complete. Sprint-status `done` corresponds to CODE GATE only. LIVE GATE is tracked as an operator-managed checklist outside sprint-status. This prevents "is the SLA actually live?" confusion after the epic closes.

- **Blameless incident process must be structurally enforced, not culturally stated.** (E21-P05) A post-mortem template structurally excluding a "who" column in the Timeline table produces blameless outputs regardless of facilitator skill. Required elements: (1) timeline entries describe system state or action (never actor name); (2) "contributing factors" section instead of "root cause" (avoids single-cause fallacy); (3) `SLA-Scope: in-scope | EXEMPT (vendor outage)` header on every runbook — prevents SLA hostage-taking to upstream vendor outages (KraftData, Stripe, ClamAV, OAuth provider). Reference: `incident-management/post-mortem-template.md` + `severity-definitions.md` (PE.06).

- **redis-py resilience hardening is mandatory for all managed Redis (ElastiCache) connections.** (E21-P06) Any service connecting to managed Redis must configure: `socket_keepalive=True`, `health_check_interval=30`, `retry=Retry(ExponentialBackoff(), 3)`, `retry_on_error=[ConnectionError, TimeoutError]`, `socket_connect_timeout=5`, `socket_timeout=10`. For Celery services: `broker_connection_retry_on_startup=True`. Without these settings, automated failover (≤10s for ElastiCache Replication Group) causes cascading connection errors. Apply to all 15 `redis.from_url()` call sites across services. Reference: PE.03 Connection Audit table.

- **`CREATE INDEX CONCURRENTLY` requires `transactional_ddl = False` in Alembic migrations.** (E21-P07) Non-concurrent index builds take an exclusive lock on the target table for minutes in production (blocking all reads). Always use `CREATE INDEX CONCURRENTLY`. The Alembic migration must set `transactional_ddl=False` in `context.configure()` — concurrent index builds cannot run inside a transaction. Template pattern from `M_PE02_opportunities_tsv_gin_index`:
  ```python
  with context.begin_transaction():
      context.configure(..., transactional_ddl=False)
      # op.execute("CREATE INDEX CONCURRENTLY ...")
  ```

---

## Anti-Patterns (Don't Do This)

### From Epic 1 Retrospective

- **Don't skip TEA automation for "simple" stories.** Stories 1.8–1.10 (CI, Helm, Terraform) were missing formal TEA automation summaries despite having tests. Every story needs a TEA review cycle.
- **Don't defer the same issue across multiple reviews without consolidating.** Docker root user was flagged in Stories 1.1 and 1.2 reviews separately. Consolidate into one tracked item.
- **Don't leave sprint-status.yaml inconsistent.** Update epic status to `done` when all stories complete, before the retrospective.
- **Don't use `ignore_missing_imports = true` globally in mypy.** Narrow to specific third-party modules to catch misspelled imports.
- **Don't bind Docker Compose ports to 0.0.0.0.** Use `127.0.0.1:<port>:<port>` to prevent network exposure.
- **Don't use `:latest` tags on infrastructure Docker images.** Pin specific versions for reproducibility.

### From Epic 2 Retrospective

- **Don't call `bcrypt.hashpw` directly in async context.** This blocks the event loop for 200–400ms, starving all concurrent requests. Always use `run_in_executor`.
- **Don't forget `User.is_active` in authentication queries.** Every query that grants a JWT (login, refresh, accept_invite) must verify the user is active.
- **Don't let Redis errors propagate as 500 from the rate limiter.** Fail-open: log the error, skip the rate limit check, continue processing. Rate limiting is a hardening layer, not a hard dependency.
- **Don't accept unbounded password input.** All password fields must have `max_length=128` (or similar) to prevent bcrypt HashDoS.
- **Don't use `HTTPException` for 429 rate-limit responses.** Use `JSONResponse` directly to produce the standard `{"error": ..., "message": ..., "details": null, "correlation_id": "..."}` envelope.
- **Don't duplicate deferred work entries across multiple code reviews.** When a cross-cutting issue (e.g., blocking bcrypt, ORM type annotations) appears in multiple stories, consolidate to one entry in deferred-work.md with references to all source stories.
- **Don't allow `isinstance(data, dict)` branch logic in response assertions.** Commit to the declared schema. Use `assert "versions" in data` — a wrong key name in production should fail the test immediately with a clear error, not silently return an empty list.
- **Don't leave `_SKIP_REASON` constants or `@pytest.mark.skip` decorators in tests after GREEN phase.** Clean up RED-phase scaffolding before marking a story done.
- **Don't defer NFR HIGH-priority items from one epic to the next without re-surfacing them.** The NFR report must include a "Carry-Forward from Previous Epic" section. Items not addressed become automatic CONCERNS in the new epic's assessment.

### From Epic 3 Retrospective

- **Don't use `user?.companyId ?? "some-stub-id"` in any component.** Stub fallback IDs silently send invalid data to real APIs at integration time. Always guard: `if (!user?.companyId) { showError(...); return; }`.
- **Don't hardcode display strings in any UI component.** Every user-visible string — labels, placeholders, error messages, empty states, breadcrumbs — must use `useTranslations()` with a key from `bg.json`/`en.json`. Hardcoded strings break BG locale.
- **Don't mark a frontend story done with incomplete TEA.** TEA automation is a hard gate, same as for backend stories. `tea_status: in-progress` blocks the epic retrospective.
- **Don't skip the redirect count assertion when modifying locale routing.** Silent redirect loops are invisible in manual testing and only surface under load or with specific browser cookie states.
- **Don't use ad-hoc conditional renders for loading/error/empty states.** This creates inconsistent UX and bypasses the unified `<QueryGuard>` + skeleton/error boundary/empty state system. Every data-fetching component uses `<QueryGuard>`.
- **Don't share Zustand persist keys between client and admin apps.** `eusolicit-auth-store` must be `eusolicit-client-auth-store` and `eusolicit-admin-auth-store`. Same-origin deployment (behind a reverse proxy) would cause the admin app to read client user tokens.
- **Don't fire multiple `POST /auth/refresh` calls concurrently.** E02's token family rotation logs the user out on second refresh. The `apiClient` singleton refresh-lock is the protection. Never bypass or replace it with a simpler retry pattern.
- **Don't use `"turbo": "latest"` in `frontend/package.json`.** Each fresh `pnpm install` may pull a different Turborepo major version (e.g., v2 → v3), silently breaking the build pipeline. Always pin to a specific minor range: `"turbo": "^2.9.5"` (or the current installed version). Check with `pnpm list turbo` and update the pin after intentional upgrades.

### From Epic 4 Retrospective

- **Don't use naive string equality (`==`) for HMAC signature comparison.** `header_sig == computed_sig` is vulnerable to timing attacks — an attacker can determine the correct signature one byte at a time by measuring response time differences. Always use `hmac.compare_digest()`.

- **Don't skip ATDD checklists for backend epics.** Epic 4 produced zero `atdd-checklist-4-*.md` files across 10 stories. The ATDD cycle (RED tests before implementation) is the discipline that proves test-driven development — skipping it makes the test design a post-hoc audit rather than a development gate. ATDD checklist scaffold must be created before story development starts, not after.

- **Don't let circuit breaker state live only in memory in multi-replica environments.** In-memory `asyncio.Lock`-guarded state is per-process. When a service scales to N replicas, each instance has independent circuit state: one instance may reject while another continues to hammer a failing agent. Document the limitation at implementation time; do not defer the Redis migration decision beyond the first auto-scaling event.

- **Don't implement SSE proxy without a wall-clock latency benchmark.** Functional tests (events forwarded in order) do not verify the < 100ms per-event latency requirement. SSE latency is a user-experience guarantee — verify it with a controlled mock SSE source with known send timestamps and `time.monotonic()` comparison before marking the story done.

- **Don't close an epic without an NFR assessment when the epic introduces stateful resilience, outbound HTTP clients, or streaming.** Epic 4 had no NFR report. The circuit breaker per-instance gap (E04-R-005), SSE latency unverified (E04-P1-009 residual), and Redis publish reliability (E04-R-003 monitoring backlog) are NFR concerns that compound over time if not formally assessed.

- **Don't defer the same security hardening items across more than two consecutive epics.** Items deferred from E01 that remained open through E04 (Prometheus /metrics, Dockerfile USER, Dependabot, bcrypt executor, User.is_active, Redis fail-open, JWT audience/issuer) are now 4 epics overdue. Each additional deferral increases integration risk. Cap deferral at 2 consecutive epics — after that, inject as a mandatory story in the next sprint.

### From Epic 9 Retrospective

- **Don't leave a story file at its pre-dev status after the dev phase completes (RC2 — 2nd recurrence).** Story 9.3's file retained `Status: ready-for-dev` with all task checkboxes unchecked despite 12 tests being implemented and passing per the ATDD checklist. The story file is the source of truth for the traceability matrix and gate decisions — a wrong status propagates to TRACE_GATE: CONCERNS. Update the status field, check all task boxes, and add a `## Dev Agent Record` section before emitting HALT. This is the same Root Cause 2 from the Story 5-5 retrospective. Layer B (`2b-dev-story-verify` phase) must be prioritised.

- **Don't defer frontend component tests without naming an activation story.** Stories 9.12 and 9.13 deferred Vitest + Testing Library component tests to "TEA phase" with no sprint or story designated to activate them. Per the E12 retrospective rule, every deferred test category must specify (1) the test file stub, (2) the story that will activate it, (3) the sprint target. "Deferred to TEA phase" alone is not acceptable.

- **Don't ship a new long-running service without Prometheus metrics.** The Notification Service deployed with ~810 tests and zero observability instrumentation. Any new service that runs periodic tasks, Celery workers, or SLA-bound operations must define its metrics in the same sprint as the service scaffold (S09.01 equivalent). Metrics are not a Sprint N+1 concern.

- **Don't leave `epic-N: in-progress` in sprint-status.yaml when all stories are `done` (4th recurrence).** `epic-9: in-progress` was set despite all 14 stories being `done`. Add a CI quality gate that enforces this — `quality-gates.yml` must fail on this condition. This has now occurred in E07, E09, E11, E12.

### From Epic 10 Retrospective

- **Don't skip epic-level test design.** Epic 10 skipped the `test-design-epic-10.md` pipeline, leaving NFR verification undefined. NFR strategy must be established during the planning phase.

- **Don't defer security middleware as a non-blocking item.** Delayed implementation of foundational security layers (e.g., S10.02) compromises the integrity of all downstream features. Security middleware is always a P0 blocking dependency.

- **Don't allow systematic TEA review under-execution.** 8 consecutive epics with 0 TEA scores indicates a failed process gate. Orchestrator must block `review -> done` transitions on TEA score presence.

- **Don't use f-string interpolation in SQLAlchemy `text()` clauses for dynamic values.** SQL injection risk invisible to ruff, mypy, and ATDD. Found in S10.09 approval decision engine: `WHERE stage_id = '{stage_id}'` instead of `bindparam("stage_id")`. Add structural test asserting bound parameters are used.

- **Don't ship Celery tasks without verifying worker registration.** A function missing `@app.task` passes all unit tests (the function exists) but is never registered in the worker process. Found in S10.03: cleanup task was callable but never registered. Add `assert task_name in celery_app.tasks` to every story's ATDD checklist.

- **Don't bundle 3+ distinct user workflows in a single frontend story.** S10.16 (approval stepper + bid/no-bid radar chart + outcome recording) generated 37 review action items (10 blocking), project record. Split at story creation time — not after review discovers the scope.

- **Don't return HTTP 201 unconditionally from upsert endpoints.** RFC 7231 requires 200 for updates. S10.10 always returned 201; found in review. Service must return `(response, was_insert)`; router maps to 201/200. ATDD must test both codes.

- **Don't let injected remediation stories sit at `ready-for-dev` across multiple epics.** `inj-01` (Dependabot), `inj-02` (k6), `inj-03` (TEA backlog) have been `ready-for-dev` for 8 consecutive epics. Injected stories with CRITICAL/HIGH severity must be P0 blockers on the next epic close gate, not optional backlog items.

- **Don't allow contradictory spec constraints to reach implementation.** S10.04 had a CHECK constraint requiring `resolved_by IS NOT NULL` when `resolved = TRUE`, combined with an FK `ON DELETE SET NULL` on `resolved_by`. The constraint is impossible to satisfy after user deletion. Resolve FK + CHECK contradictions in story creation, not mid-implementation.

### From Epic 5 Retrospective

- **Don't use a module-level dict (`_RUN_ID_REGISTRY`) to correlate Celery task IDs with DB record IDs in signal handlers.** Celery prefork pool fires `task_failure` signals in a different OS process than the task body. The in-memory dict is not shared across processes — `_pop_run(task_id)` returns `None`, the audit record (`crawler_runs.status`) is never marked `failed`, and orphaned `running`/`retrying` rows accumulate silently. Tests pass because they use eager mode (single process). Always use Redis SETEX/GETDEL for signal-handler ↔ task-body state correlation.

- **Don't use separate DB sessions for audit-record creation, data upsert, and audit-record status update within the same Celery task.** Three separate `get_sync_session()` contexts create a partial-failure window: if the process crashes between the second and third session, data is persisted but the audit record remains in `running` forever with no automatic recovery. Minimise session boundaries; add a startup recovery sweep task to reclaim stuck records.

- **Don't skip ATDD checklists for backend epics — 3rd recurrence (E04, E05, and prior).** E05 produced zero `atdd-checklist-5-*.md` files despite E04's critical commitment. The cultural norm is insufficient — the orchestrator must mechanically block the `2-dev-story` phase from starting without a checklist file present. ATDD is a development gate, not a post-hoc audit.

- **Don't leave a story file at its pre-dev/pre-review status after the story completes (RC2 — 3rd recurrence).** Story 5.5 file retained `Status: ready-for-review` while sprint-status.yaml shows `5-5: done`. Downstream traceability tools treat the story file as the source of truth. A wrong status propagates to TRACE_GATE: CONCERNS and misleads the orchestrator. Update `Status:`, check all task boxes, and add the `## Dev Agent Record` section before emitting HALT. The `2b-dev-story-verify` orchestrator phase is the mechanical fix.

- **Don't design circuit breakers with a broad `except Exception` catch that counts 4xx responses as failures.** `circuit_breaker.py` with a generic `except Exception` handler that increments the failure counter for all exception types will prematurely open the circuit on repeated 400 responses (e.g., misconfigured normalization payload). Distinguish retriable errors (5xx, transport, timeout) from non-retriable client errors (4xx) at the circuit-breaker level. Only the former should increment the failure count.

- **Don't read env vars for batch-size limits without guarding against 0 or negative values.** `batch_size = int(os.environ.get("ENRICHMENT_BATCH_SIZE", "20"))` with no guard silently processes nothing when `ENRICHMENT_BATCH_SIZE=0`. Always apply `batch_size = max(1, batch_size)`.

### From Epic 12 Retrospective

- **Don't mark a story `done` if it has frontend ACs and no E2E spec file.** Absence of a spec file means the AC is unverified at the browser level. A backend API test for a frontend AC is not sufficient — it proves the API works, not that the UI renders correctly.
- **Don't treat a non-functional story (load test, security audit) as done when its output files contain only templates.** Placeholder dashes in load-test-results.md and unchecked boxes in security-audit-checklist.md are not evidence. These stories have hard output artifact requirements.
- **Don't generate Celery tasks without integration test coverage of their execution.** Testing the endpoint that enqueues a task is not the same as testing the task. The side effects (S3 upload, DB write, SendGrid call) must be verified end-to-end.
- **Don't leave `test.skip()` decorators in E2E specs at epic close without a named activation story.** RED-phase specs are a deliverable; their activation plan is a separate required deliverable.
- **Don't skip TEA reviews because a story is "fully covered" or "simple".** Test quality and isolation can only be validated by a formal TEA pass. High test counts do not guarantee test quality.
- **Don't leave `epic-N: in-progress` in sprint-status.yaml when all stories are `done`.** This is the 4th occurrence of this anti-pattern. It will be enforced by CI going forward.
- **Don't implement analytics queries without `WHERE company_id = :cid` scoping on every materialized view query.** Cross-tenant data leakage in analytics is an R12.1 (Score 6) risk. Every analytics service method must include a cross-tenant negative test in its test suite.
- **Don't use `REFRESH MATERIALIZED VIEW` without `CONCURRENTLY` on production tables.** Non-concurrent refresh takes an exclusive lock that blocks all reads during refresh. Always add a `UNIQUE` index on materialized view columns used for filtering before enabling `CONCURRENTLY`.

### From Epic 6 Retrospective

- **Don't implement SSE endpoints without the four generator lifecycle checks.** The following four bugs will reach production if not explicitly guarded: (1) `asyncio.shield(semaphore.acquire())` causes permanent permit leaks on timeout — use `await asyncio.wait_for(semaphore.acquire(), timeout=X)` directly; (2) usage/quota check not done before `StreamingResponse` creation — 429 inside generator cannot change already-committed 200 headers; (3) `asynccontextmanager` stream generator not closing inner async generator in `finally` — httpx resources leak on consumer cancellation; (4) no terminal event emitted when upstream stream ends without `done`/`error` — client is left with orphaned connection. Add all four to the SSE story spec checklist.

- **Don't deploy frontend components that consume upstream query hooks without declaring the exact query key.** A query key mismatch (e.g., `["opportunity-detail", id]` vs. the upstream `["opportunity", id]`) causes `invalidateQueries` to silently fail — the cache is never invalidated, stale data is displayed, and no error is raised. Story specs for frontend components consuming upstream hooks MUST include a section "Required query keys from upstream stories" with the exact key array copied from the source story.

- **Don't mark a story `done` unless ALL three completion gates pass.** The three required gates are: (1) Senior Dev Review APPROVED (not just "patches applied"), (2) ALL ATDD tests GREEN (not "tests written but skipped"), (3) i18n parity check passing (`pnpm check:i18n` shows no missing keys). A story meeting only two of three gates is `in-review`, not `done`. This applies equally to backend and frontend stories.

- **Don't generate traceability matrices before the implementation spec (story file) is locked.** A traceability matrix generated concurrently with story authoring can diverge from the spec (e.g., 100MB vs 50MB file size limit). The correct sequence is: lock story spec → generate ATDD checklist → implement → generate traceability matrix from the ATDD checklist. ATDD checklists are the source of truth; traceability derives from them.

- **Don't specify Redis-gated features without an explicit fail-open requirement.** If Redis is unavailable, a Redis-gated feature (UsageGate, rate limiter) must allow the request through, not fail it. Template: "If Redis is unavailable, allow the request, log `structlog.warning('redis_gate_degraded')`, and set `X-{Feature}-Remaining: -1`." This rule already applies to the rate limiter (E02); extend it to every new Redis-gated feature.

- **Don't carry the same infrastructure blocker past a 4th epic.** Dependabot, Prometheus /metrics, and k6 baseline have each been deferred for 4+ consecutive epics. The rule: any infrastructure item deferred twice becomes a mandatory story in the next sprint. A 4th deferral is a process failure — escalate to the operator and block the relevant milestone gate.


### From Epic 7 Retrospective

- **Don't maintain frontend TypeScript types manually — generate from backend OpenAPI spec.** Manually maintained frontend type files diverge from backend Pydantic schemas. `ProposalResponse` was missing `current_version_number` and `generation_status`, only caught in TEA review. Frontend types MUST be generated via codegen (`openapi-typescript`). Manual type duplication is prohibited for any backend-derived response type.

- **Don't omit numeric constants from story acceptance criteria.** Breakpoint thresholds (e.g., 1024px vs 1280px), file size limits, timeouts, and retry counts must be listed in an "Implementation constants" subsection of each story AC. Undeclared constants cause silent implementation divergence only caught in TEA review.

- **Don't mark a story `done` if any P0 AC has 0% confirmed-passing (GREEN) tests.** Stories with P0 ACs that have only RED/written tests cannot be `done`. A story is `in-review` until ≥1 GREEN test per P0 AC is confirmed. This is the per-story gate that prevents TRACE_GATE failures where test existence is 100% but quality assurance is zero.

- **Don't state ATDD test count totals that diverge from level-by-level breakdowns.** A header total that doesn't match the sum of individual levels creates misleading coverage metrics. Add an auto-computed total to the ATDD checklist template; CI lint should validate the summary total matches the breakdown.

### From Epic 8 Retrospective

- **Don't treat retrospective [ACTION] items as advisory.** E07 retro designated k6 baseline, Dependabot, and S07.04 R-001 optimistic locking GREEN verification as Epic 8 hard deliverables. None were completed. Every `[ACTION]` item with `SEVERITY: critical` must have an Orchestrator verification task checked at the next retrospective before new items are accepted. Unresolved critical carry-forwards must block epic kickoff.

- **Don't classify E2E specs with all tests wrapped in `test.skip()` as PARTIAL coverage — they are NONE coverage.** A spec file where ≥80% of tests are `test.skip` consumes engineering time without providing quality assurance. ATDD checklists must include E2E test activation as a blocking sub-task before a story is marked `done`.

- **Don't deploy Stripe or any outbound payment SDK calls without circuit-breaker protection.** The E04 two-layer resilience pattern (`circuit_breaker(retry(http_factory))`) is the standard for ALL outbound HTTP. Simple `try/except` logging does not prevent cascading failures on a degraded external endpoint. All outbound payment API calls must adopt this pattern.

- **Don't deploy revenue-critical paths without Prometheus metrics.** Billing webhook processing latency, usage sync drift, Stripe API error rate, per-tier subscription counts, and trial-to-paid conversion rate are required operational metrics. A billing failure invisible until a user complains is a platform reliability failure.

### From Epic 12 Retrospective (Supplementary — 2026-04-25)

- **Streaming CSV export is mandatory for admin endpoints returning potentially large datasets (>10k rows).** Use `StreamingResponse(generate_csv_rows(query), media_type="text/csv")` — never load all rows into memory before serialising. Prevents OOM on large audit log, platform analytics export, or reporting endpoints. Include a test asserting `Content-Type: text/csv` and that the response is a generator (not a pre-buffered list). Reference: S12.13 audit log export.

- **Previous epic TRACE_GATE failures must be verified as resolved before the subsequent epic retrospective gate.** When an epic closes with TRACE_GATE FAIL (e.g., E11 at 75% P1 coverage due to missing S11.07 backend endpoints), the next epic must include an explicit entry criterion confirming the failed ACs are now covered. Carrying P1 AC gaps forward without acknowledgment invalidates the traceability of all downstream stories that build on the missing features.

- **Pre-story entry criteria for non-functional stories (load test, security audit) must confirm staging environment and tooling availability before the story begins.** Required checks at story kickoff: (1) staging confirmed running with production-representative data volume (≥500k rows for analytics load tests), (2) tooling (k6/Locust, ZAP, LocalStack) confirmed installed and configured, (3) output artifact templates exist marked "pending results". Story cannot close `done` with placeholder content in output files — actual results are required.

### From Epic 13 Retrospective

- **Don't let a new dev session overwrite code approved in a prior review pass.** Before editing any file that has prior review findings, re-read the current file state and list all resolved deviations that must be preserved. E13: a new session modified `proposals.py` for unrelated AC18 work, overwrote the fire-and-forget audit write (F4 resolved in 2nd review pass) with a synchronous dual-write, violated AC3, and caused 3 integration tests to fail. Add to dev-story template: "Before editing a file with prior review findings, re-read and list all resolved deviations to preserve." (AP13-01, SEVERITY: critical)

- **Don't approve a review that claims "tests pass" without quoting the actual test execution output line.** Any review that approves a previously-failing test criterion MUST include the real pytest summary line (e.g., `3 failed, 16 passed in 12.4s`). A review without a quoted execution result is invalid — it may trust the agent's summary over actual HEAD state. E13: the 2nd-pass review approved "test_audit_log_row_has_correct_shape — PASS"; the 3rd pass re-executed and found 3 failures. (AP13-02, SEVERITY: critical)

- **Don't write `done` in sprint-status.yaml when the story file status is still `review` with blocking findings.** The orchestrator's story-close workflow must cross-check the story file `Status:` field before writing `done` to sprint-status.yaml. A sprint-status `done` + story file `review` discrepancy propagates a false completion signal to all downstream tools (retro, NFR assessment, traceability). (AP13-03, SEVERITY: high)

- **Don't leave AC text showing the original spec when the implementation chose a different schema.** Any implementation-time schema deviation from an AC must update the AC text in the story file before the review approves. The AC text is the contract; divergence causes future reviewers to rediscover the same "spec violation" repeatedly. E13: AC1 specified `{"error":"ai_gateway_error","correlation_id":"..."}` but implementation chose `{"error":"<prose>","code":"gateway_error","correlation_id":"..."}` — neither AC text nor tests were updated. (AP13-04, SEVERITY: medium)

- **Don't append injected `inj-*` carry-forward stories to the backlog behind feature work.** Critical injected stories (Dependabot, k6, TEA reviews) must be the FIRST items executed in the next epic's queue, not the last. In E13, all 7 injected stories sat at `ready-for-dev` while Epic 14 advanced 3 stories. This is the same failure mode documented in E08 retro: "retro-to-action feedback loop is broken." The orchestrator must enforce `inj-*` execution order as p0. (AP13-05, SEVERITY: critical)

- **Don't leave a story's `Epic:` header field misaligned with its sprint-status assignment.** When a story is injected or reassigned to an epic, update its `Epic:` header field immediately. Misaligned headers (e.g., `Epic: Post-E07 cleanup` for a story tracked under E13) confuse traceability tools, cause incorrect AC scope dating, and mislead agents reading the story context. (AP13-06, SEVERITY: medium)

- **Don't use bare `str` for Pydantic response schema fields with known value sets.** Use `Literal["draft", "active", "archived"]` or a `StrEnum` subclass. Bare `str` prevents OpenAPI enum schema generation, allows unknown values to reach the frontend without validation errors, and loses machine-readable contract value. Add to backend story template and ATDD checklist. (AP13-07, SEVERITY: medium)

- **Don't use `asyncio.get_event_loop()` in Celery tasks bridging sync→async.** It is deprecated in Python 3.12 and raises `RuntimeError` in 3.14+. Correct pattern: `loop = asyncio.new_event_loop(); try: result = loop.run_until_complete(async_fn()) finally: loop.close()`. Never `asyncio.run()` (fails when a loop is already running). Add to Celery task template and ATDD checklist. (AP13-08, SEVERITY: medium)

### From Epic 19 Retrospective

- **Don't reimplement aggregation logic in secondary service paths.** When a canonical aggregation function exists (e.g., `reporting_service.aggregate_outcomes()`), all code paths — scheduled, on-demand, webhook-triggered — MUST call that function. Reimplementing equivalent logic inline produces output divergence that is invisible in unit tests but surfaces as data inconsistency in E2E or production. Fence: grep `services/` for any new `_aggregate_*` or `_compute_*` helper before writing it, and verify no canonical implementation already exists. (AP19-C4, Source: E19 S19-2 B-7.)

### From Epic 21 Retrospective

- **Don't start a non-functional (load test) story without confirming staging environment availability and production-representative data volume.** (AP21-01, SEVERITY: high) PE.01's FTS baseline was captured against a local 10K-row dataset because staging was unavailable (stale service image blocking rebuild). Throughput numbers (requests/s, p95 at load) are unvalidated at production scale. Entry criteria for any load-test story: (1) staging environment confirmed running with current service images; (2) ≥100K production-representative rows in target tables; (3) tooling (k6, output artifact templates) confirmed ready. A story cannot close `done` with only local-environment baselines without a documented deviation and a named follow-up operator validation step.

- **Don't write any story with FTS performance assertions without first running `EXPLAIN ANALYZE` to confirm the query plan.** (AP21-02, SEVERITY: high) PE.01 AC-2.4 discovered a Seq Scan on the `tsv TSVECTOR` column (289ms at 10K rows, projecting to ~28s at 1M rows) during Pass-7 B10 — after multiple dev passes. The gap: story planning assumed a GIN index existed. Pre-story entry criteria for any FTS story: "Run `EXPLAIN ANALYZE <fts_query>` on the target table; confirm Bitmap Index Scan (not Seq Scan). If Seq Scan: create `CREATE INDEX CONCURRENTLY` migration before measuring performance." This check takes 30 seconds; omitting it cost PE.01 a revision pass and PE.02 a mandatory schema migration.

- **Don't treat TEA reviews as a separate backlog story (`inj-*`) — embed as a mandatory AC in the bmad-dev-story template.** (AP21-03, SEVERITY: critical — 16th consecutive epic without TEA score) The `inj-03` TEA review backlog story has been at `ready-for-dev` for 16 consecutive epics (E08–E21). The failure mode is systemic: a separate backlog item is never prioritized over feature/infra stories. The only mechanical fix: add TEA review as a required AC in the `bmad-dev-story` phase template. Format: "TEA review executed; score ≥ 80/100 documented in Dev Agent Record; or blocking findings enumerated with remediation plan." Remove `inj-03` from the backlog — the separate-backlog strategy does not work.

- **Don't rely on the dev agent to self-update story file `Status:` — implement orchestrator AP18-C2 atomic patch mechanically.** (AP21-04, SEVERITY: critical — 18th consecutive epic with Status mismatch) All 6 E21 story files show `Status: review` after dev passes complete. This is the same AP20-C1 / AP16-03 / AP13-03 / AP13-02 failure (documented ≥5 times). Root cause: not dev agent behavior — it is an orchestrator configuration gap. The `2b-dev-story-verify` phase has never been implemented to cross-check story file `Status:` against sprint-status `development_status` before emitting HALT. Escalate to operator: implement this verification as a mechanical gate. Until then, every story in every epic will exhibit this mismatch.

---

## Technology Stack & Versions

- **Python:** 3.12+ (all services). Use modern syntax (`type` statements, `match/case`).
- **Framework:** FastAPI with Starlette middleware pattern
- **ORM/DB:** SQLAlchemy (async), Alembic migrations, asyncpg, PostgreSQL 16
- **Cache/Events:** Redis 7 via redis-py (Redis Streams, no Celery)
- **Models:** Pydantic v2 with `BaseSettings` for config
- **Linting:** ruff (I/E/W/F/UP rules), mypy (Python 3.12 target)
- **Testing (backend):** pytest + pytest-asyncio (asyncio_mode="auto"), pytest-cov, Factory Boy
- **Frontend:** Next.js 14 App Router, TypeScript strict, pnpm workspaces + Turborepo
- **Frontend testing:** Vitest (unit/component), Playwright (E2E/browser)
- **Frontend state:** Zustand (persist + devtools middleware), TanStack Query v5
- **Frontend forms:** React Hook Form + Zod (`useZodForm`), shadcn/ui primitives
- **Frontend i18n:** next-intl v3 (`app/[locale]/` route structure, `locales: ['bg', 'en']`, `defaultLocale: 'bg'`)
- **UI components:** shadcn/ui in `packages/ui` (46+ components, barrel-exported)
- **Infra:** Docker Compose (local), Helm 3, Terraform >= 1.5, GitHub Actions CI
- **Logging:** structlog (JSON production, pretty-print dev)

---

## Critical Implementation Rules

### Database

1. **Schema isolation is enforced.** Each service has its own PostgreSQL schema. Service roles have CRUD only on their own schema + SELECT on `shared`. Never write cross-schema queries from application code.
2. **Use migration_role for DDL only.** Application code must not use migration_role connections.
3. **Always specify `schema=` in Alembic migrations.** The mako template lacks a SCHEMA constant — every `op.create_table()` call must explicitly pass the schema parameter. Omitting it creates tables in the wrong schema.
4. **Alembic migration naming:** `NNN_descriptive_name.py` (sequential, not timestamp-based).
5. **version_table_schema** is set per-service in env.py to avoid alembic_version collisions.

### Redis Streams

6. **Use EventPublisher/EventConsumer from eusolicit-common.** Do not use raw Redis commands for event bus operations. The abstraction handles envelope creation, DLQ, and correlation IDs.
7. **Event envelope is mandatory:** `{event_type, payload, timestamp, correlation_id, source_service}`. All events must include all fields.
8. **DLQ is at-least-once.** The xadd→xack sequence is non-atomic. Do not assume exactly-once delivery from DLQ. (Lua script upgrade planned.)

### Shared Packages

9. **All services must import from eusolicit-common** for config, logging, middleware, health, and exceptions. No direct `print()` or `logging.getLogger()`.
10. **Inter-service DTOs live in eusolicit-models.** Event schemas use `Literal` type discriminators for pattern matching.
11. **eusolicit-kraftdata is types-only.** No HTTP client code — only Pydantic models matching the KraftData API.

### Docker & CI

12. **Docker images use multi-stage builds.** Target size < 200MB per service.
13. **Shared packages installed via relative path references** in pyproject.toml (`path = "../../packages/eusolicit-common"`).
14. **CI matrix builds all 8 projects in parallel** (5 services + 3 packages). Each runs ruff, mypy, pytest.

### Testing

15. **Use eusolicit-test-utils** for all test fixtures: DB sessions, Redis clients, JWT generators, Factory Boy factories, API clients.
16. **Test levels:** unit (no external deps), smoke (file/config validation), integration (live Docker services), E2E (Playwright).
17. **Markers:** `@pytest.mark.unit`, `@pytest.mark.integration`, `@pytest.mark.smoke`, `@pytest.mark.slow`, `@pytest.mark.cross_service`.
18. **Test isolation:** Per-test transaction rollback for DB, flush for Redis. No shared mutable state.

### Frontend Architecture

19. **All UI components live in `packages/ui`.** Never create app-local components that duplicate shared functionality. Add to `packages/ui/src/components/` and barrel-export via `packages/ui/src/index.ts`.
20. **Use `<AppShell>` for the authenticated layout.** Both client and admin apps use the shared `<AppShell>` component with sidebar/topbar/children slots. Never rebuild the shell in a feature component.
21. **Data fetching must use `<QueryGuard>`.** All list/table/detail views wrap TanStack Query state with `<QueryGuard isLoading={...} isError={...} isEmpty={...}>`. No ad-hoc `if (isLoading) return <Spinner />` patterns.
22. **Forms use `useZodForm(schema)` + `<FormField>`.** Never use `useForm()` directly. Define the Zod schema in `lib/schemas/`, call `useZodForm(schema)`, and compose `<FormField>` for every input. This wires RHF + zodResolver + inline error display + BG/EN translations automatically.
23. **Auth guard is dual-layer.** Protected routes use `<AuthGuard>` in `app/[locale]/(protected)/layout.tsx` (client-side; shows full-page spinner until `onFinishHydration()`) AND `middleware.ts` (server-side; checks `eusolicit-session` cookie, 307 redirect). Never protect a route with only one layer.
24. **AuthGuard has a 5-second hydration timeout.** If `onFinishHydration()` does not fire within 5 seconds (corrupt localStorage), treat as unauthenticated and redirect to `/login`. Prevents permanent spinner.
25. **Zustand persist keys are namespaced per app.** Use `eusolicit-client-auth-store` and `eusolicit-admin-auth-store` (not a shared key). Prevents cross-app localStorage collision if apps are deployed on the same origin.
26. **`apiClient` uses a Promise singleton refresh-lock for 401s.** All concurrent 401 responses queue until the first token refresh resolves, then retry with the new token. Never call `POST /auth/refresh` more than once per concurrent 401 burst — this triggers E02's token family revocation.
27. **Add `x-request-id: crypto.randomUUID()` to every apiClient request.** This enables correlation between frontend errors and backend logs when observability is operational.
28. **Locale routing uses `localePrefix: 'always'`** with `locales: ['bg', 'en']` and `defaultLocale: 'bg'`. All routes are prefixed: `/bg/dashboard`, `/en/dashboard`. Test that each entry path triggers ≤1 redirect hop using a `countRedirects()` utility in E2E tests.
29. **All UI strings use `useTranslations()` / `getTranslations()` from next-intl.** Never hardcode display strings in components. All namespaces are: `common`, `nav`, `auth`, `forms`, `errors`, `wizard`. BG/EN message files must have identical key sets — enforced by the key-parity check test.
30. **Shell components are server-compatible wrappers.** `<AppShell>`, `<Sidebar>`, `<TopBar>` are server components. Only the interactive leaves (`<SidebarToggle>`, `<UserAvatarMenu>`, `<LanguageSelector>`) carry `'use client'`. Keep this boundary discipline in all feature components.
31. **Wizard state persists in Zustand across page reload.** The `wizardStore` uses `persist` middleware. If `user?.companyId` is undefined at wizard submission, show a validation error — never fall back to a hardcoded stub ID.
32. **Pages using `useSearchParams()` must wrap the component in a `<Suspense>` boundary.** Next.js App Router throws a build error if `useSearchParams()` is called outside a `<Suspense>` boundary. The page default export renders `<Suspense>` wrapping the inner `"use client"` component that reads search params. See `apps/client/app/[locale]/(auth)/callback/page.tsx` as the reference implementation. Applies to any page reading URL state: OAuth callbacks, pagination, filters, redirect params.

### AI Gateway & External Service Integration

47. **All outbound HTTP calls to external services must use the two-layer resilience pattern.** Circuit breaker (outer, per-logical-name) wraps retry (inner, exponential backoff). Never call an external API with only retry or only circuit breaking. The `call_kraftdata()` function in `ai-gateway` is the reference implementation.
48. **Webhook signature validation must use `hmac.compare_digest()`, never `==`.** Applies to all inbound webhooks regardless of provider. Always read the raw request body bytes BEFORE parsing JSON (body bytes are required for HMAC). Unit test must verify both `hmac.compare_digest` is used in source AND that timing difference between valid/invalid signatures is < 1ms.
49. **Agent/service registry YAML must fail fast on startup.** If `config/agents.yaml` is missing, has duplicate entries, or fails Pydantic validation, the service must raise an error and refuse to start. Never silently ignore registry loading failures — this would cause all downstream AI calls to return 404.
50. **Streaming endpoints (`run-stream`) must enforce idle timeout (120s), total timeout (600s), and heartbeat (15s).** Without heartbeat, proxies/load balancers may drop idle SSE connections with no signal to the client. Without timeouts, an abandoned stream holds a semaphore permit indefinitely, blocking subsequent requests.
51. **SSE `StreamingResponse` requires `Cache-Control: no-cache` and `X-Accel-Buffering: no`.** These headers prevent nginx and CDN intermediaries from buffering SSE chunks. Always set on any endpoint returning `text/event-stream`.
52. **`X-Caller-Service` header is required on all AI Gateway execution endpoints.** Used for execution logging, rate limit attribution, and debugging. Return 400 with a descriptive error if missing. Generating services must always pass this header when calling the gateway.

### Authentication & Security

32. **JWT middleware validates only RS256 signature + expiry by default.** Add `audience` and `issuer` claims to `jwt.decode()` calls if multiple services share the same RSA key pair.
33. **`User.is_active` must be checked in every authentication query.** Both `login()` and `refresh_token()` must filter `User.is_active == True`. Deactivated users must not receive new tokens.
34. **Wrap `bcrypt.hashpw` in `run_in_executor`.** `bcrypt` at cost=12 takes 200–400ms and blocks the async event loop. Use `await loop.run_in_executor(None, bcrypt.hashpw, ...)` in every password-setting flow.
35. **All password fields must have `max_length=128`.** Prevents bcrypt HashDoS. Applies to `LoginRequest`, `RegisterRequest`, `PasswordResetConfirmRequest`, `AcceptInviteRequest`, and any future password field.
36. **Rate limiter must fail-open on Redis `ConnectionError`.** Catch the exception, log it as structured warning, and continue processing. Redis outage must not block authentication.
37. **Refresh token rotation requires pessimistic locking.** Use `SELECT ... FOR UPDATE` on the refresh token row to prevent concurrent rotation from forking the token family.
38. **Every company-scoped endpoint must include a cross-tenant negative test.** `Company A credentials → Company B resource → 403`. No exceptions.
39. **OAuth callback must validate the `state` parameter against a session-stored nonce.** Missing or mismatched `state` is a CSRF failure — reject with 400, do not default to empty string. Required before any stub replacement with a real OAuth provider.
40. **`dangerouslySetInnerHTML` usage requires an explicit security review comment.** Any component rendering user-generated content with `dangerouslySetInnerHTML` must include a comment explaining the sanitization applied. Flag in all code reviews from E07 (proposals) onwards.

### RBAC

41. **RBAC uses two-tier enforcement:** company role ceiling (5 levels: admin > bid_manager > contributor > reviewer > read_only) + entity-level permissions (read/write/manage). The role ceiling is always the upper bound; entity permission cannot exceed it.
42. **Admin and bid_manager bypass entity-level permissions for their own company only.** The bypass check must verify `company_id` match before granting access.
43. **`entity_permissions` rows must have UNIQUE constraint on `(user_id, company_id, entity_type, entity_id, permission)`.** Without it, duplicate rows cause `MultipleResultsFound` → 500. Enforce at application layer until DB migration adds the constraint.

### Audit Trail

44. **All mutations (POST/PUT/PATCH/DELETE) must write to `shared.audit_log`.** Capture `entity_type`, `entity_id`, `action_type`, `before` (None for creates), `after` (None for deletes), `user_id`, `ip_address`.
45. **Audit writes must be non-blocking.** Wrap in `try/except`; suppress failures silently (log, do not raise). A failed audit write must never return 500 to the client.
46. **`ip_address` from `request.client.host` is proxy IP behind a reverse proxy.** Future work: read from `X-Forwarded-For` header with proxy whitelist validation.

---

## Patterns (Do This)

### From Epic 1 Retrospective

- **ATDD-first for infrastructure/security stories.** Write failing acceptance tests before implementation. This produced 420 tests for schema isolation (Story 1.3) and caught real issues.
- **Risk-driven test investment.** Allocate more tests to high-risk items (Score ≥ 6). Stories 1.3 and 1.4 got 140 and 246 tests/point respectively.
- **Deferred work tracking with rationale.** Every code review item that isn't fixed immediately goes to `deferred-work.md` with source story, description, and reason for deferral.
- **Regression verification per automation round.** Each TEA automation run must verify the full test suite, not just new tests.
- **Shared test utilities early.** Create reusable fixtures in eusolicit-test-utils before writing feature tests.

### From Epic 2 Retrospective

- **Transaction-rollback fixture pattern for API integration tests.** Every API test fixture must use `await session.rollback()` in a `finally` block alongside `fastapi_app.dependency_overrides.clear()`. This is the gold standard for per-test DB isolation confirmed by TEA (Story 2.12: Isolation 100/100).

  ```python
  # Canonical pattern — replicate for every new API test fixture
  async with client_api_session_factory() as session:
      fastapi_app.dependency_overrides[get_db_session] = override_db
      try:
          yield client, session, access_token, company_id_str
      finally:
          fastapi_app.dependency_overrides.clear()
          await session.rollback()
  ```

- **Deterministic role-injection helper (`_register_and_verify_with_role`).** Null out TempCo membership's `accepted_at` before inserting the target membership to ensure `auth_service.login()` deterministically encodes the target company and role in the JWT. Reuse verbatim in any test verifying role-based authorization.

- **Exhaustive parametrized permission matrix for RBAC stories.** Cover all N roles × M permissions × own-company/cross-company combinations with `@pytest.mark.parametrize`. No partial coverage of the permission matrix is acceptable.

- **Cross-tenant negative tests are mandatory for every company-scoped endpoint.** Register Company A and Company B in the same rollback-scoped session. Verify Company A credentials return 403 on all Company B resources.

- **Parametrize >2 structurally identical tests.** If two or more test methods differ only in one variable (e.g., missing field name, role name, error code), use `@pytest.mark.parametrize`. The project already uses this pattern — apply it consistently.

- **Extract cross-story test helpers to eusolicit-test-utils at the first reuse.** Don't wait for the third story to discover the pattern; extract shared helpers as soon as the second story needs them.

- **Contribute all mutations to the shared `shared.audit_log`.** New stories introducing POST/PUT/PATCH/DELETE endpoints must write audit entries. Audit writes are non-blocking (try/except).

### From Epic 3 Retrospective

- **`<QueryGuard>` is mandatory for list/table/detail views.** Every component that fetches data with TanStack Query must wrap its state transitions in `<QueryGuard>`. Ad-hoc `if (isLoading) return <Skeleton />` scattered through feature components is the anti-pattern this replaces.

- **`useZodForm(schema)` + `<FormField>` is the only form construction pattern.** Define the Zod schema first, call `useZodForm(schema)`, compose `<FormField>` variants. This gives you zodResolver, inline animated error display, BG/EN i18n translation, and RHF controller wiring for free.

- **All new feature UI goes into `packages/ui`.** Do not create app-local `components/` folders for shared elements. If a component is used by both client and admin apps, or might be in the future, it belongs in `packages/ui`.

- **Locale routing changes must include a redirect count assertion.** Any PR touching `middleware.ts` or `i18n.ts` must include a Playwright test asserting ≤1 redirect hop for the affected routes, using the `countRedirects()` pattern from `locale-redirect.spec.ts`.

- **Produce a Stub Removal Checklist alongside any stub implementation.** When a story introduces MSW mocks, hardcoded IDs (e.g., `stub-company-id`), or stub API responses, the story deliverables must include a checklist documenting every stub and which epic/story will replace it. This prevents silent integration failures.

- **Dual-layer auth guard pattern is established — apply it without deviation.** `<AuthGuard>` (client-side spinner until hydration) + `middleware.ts` (server-side cookie check) is the standard. New routes in the `(protected)` group automatically inherit the client guard. Ensure `middleware.ts` matchers cover any new route paths.

- **Toast notifications via `useToast()` from `packages/ui`.** Call `toast.success(title)`, `toast.error(title)`, `toast.info(title)` — not custom alert patterns. The toast system handles stacking, auto-dismiss timers, and portal rendering.

### From Epic 4 Retrospective

- **All outbound HTTP calls use a two-layer resilience pattern: circuit breaker (outer) wrapping retry (inner).** `circuit.call(lambda: with_retry(http_factory, agent_name=...))` — circuit rejects immediately when OPEN; retry applies exponential backoff (1s → 2s → 4s, ±25% jitter) for 5xx/timeout/connection errors only. 4xx and `CircuitOpenError` are non-retryable and propagate immediately through both layers. Apply this pattern to any new outbound HTTP client in subsequent epics.

- **YAML-driven registry pattern decouples business logical names from external provider IDs.** `config/<service>.yaml` maps logical names to UUIDs/identifiers/types; loaded at startup via Pydantic models; fails fast on missing/duplicate entries; supports hot-reload via admin endpoint guarded by `asyncio.Lock`. Use for any integration where external IDs may be rotated independently of business logic.

- **All inbound webhook signatures must use `hmac.compare_digest()` — not string equality (`==`).** Naive string comparison is vulnerable to timing attacks. Import from stdlib `hmac` module. Unit test must inspect source code for `compare_digest` usage AND verify constant-time behavior with an off-by-one-byte signature under a timing assertion (< 1ms difference). Required for any webhook endpoint regardless of perceived risk.

- **Audit and logging writes must use `asyncio.create_task()` (fire-and-forget).** DB writes for audit trails, execution logs, and webhook logs must never block the response path. Wrap in `asyncio.create_task(_write_to_db(...))`. The handler must catch all DB exceptions and emit a structured ERROR log with enough context (execution_id, entity_id) for manual recovery. Logging failure must NOT propagate a 500 to the caller.

- **testcontainers + respx is the mandatory backend integration test stack.** testcontainers provides real PostgreSQL and Redis instances; `respx.MockRouter` mocks all outbound HTTP (KraftData, external APIs); `httpx.AsyncClient(transport=ASGITransport(app, lifespan="on"))` runs the full FastAPI stack in-process. This combination is CI-safe (no live credentials), deterministic (no network flakiness), and exercises the complete request lifecycle including DB writes and Redis publishes. All backend epics must produce an integration test suite using this stack.

- **Admin introspection endpoints are mandatory for all stateful backend services.** Every service that owns stateful resilience mechanisms (circuit breaker, rate limiter, connection pool, async queue) must expose ClusterIP-only admin endpoints: `GET /admin/circuits`, `GET /admin/rate-limit`, `GET /admin/executions`. These are operational observability surfaces — required as acceptance criteria in any story implementing stateful components.

- **SSE streaming responses require `Cache-Control: no-cache`, `X-Accel-Buffering: no` headers.** Without these, nginx/CDN proxy layers buffer SSE chunks and the client never receives events in real time. Always set these response headers on `StreamingResponse` for SSE endpoints. Also forward `X-Request-ID` in response headers.

- **Per-instance (in-memory) circuit breaker state is single-replica only.** Acceptable for initial deployment. When auto-scaling is planned, circuit state must be migrated to Redis so all replicas share the same failure count and cooldown state. Document this limitation explicitly at implementation time; create a pre-scale backlog item.

- **`asyncio.Semaphore` for concurrency control must track active/queued/rejected counts for observability.** Rate limiter state (active requests, queued requests, total rejected) must be exposed via an admin endpoint. Streaming requests hold their semaphore permit for the full stream duration — design for this in capacity planning (streaming load starves sync calls).

- **Integration test mocks must assert the exact Authorization header format, not just that a request was made.** When using respx (or any HTTP mock library) to mock outbound calls to external providers, always assert the exact auth scheme with `.match(headers={"Authorization": re.compile(r"Bearer .+")})` or equivalent. A wrong auth scheme (e.g., `X-API-Key` instead of `Authorization: Bearer`) survives the entire integration test suite and is only caught in production. This applies to all outbound HTTP mocks — KraftData, Stripe, SendGrid, Google OAuth — and to any PR that refactors HTTP client configuration. (Source: E04 S04.02 critical review-fix, 2026-04-23.)

### From Epic 9 Retrospective

- **Dual-layer consumer idempotency: DB constraint + Redis SETNX.** For all Celery Redis Stream consumers, use `INSERT ... ON CONFLICT DO NOTHING` on a uniqueness constraint at the DB layer AND a `SETNX` per-recipient key at the Redis layer. This prevents duplicate email dispatch under at-least-once delivery even during crash-before-ACK scenarios. Test both layers independently and together.

- **`asyncio.to_thread` wrapper is mandatory for Celery `.delay()` calls inside async consumers.** When a Celery task is dispatched from within an `async` def consumer (e.g., a Redis Stream consumer running in the asyncio event loop), wrap the synchronous `.delay()` call: `await asyncio.to_thread(task.delay, ...)`. Without this, the synchronous Celery call blocks the event loop. Add a test asserting `asyncio.to_thread` is called rather than `.delay()` directly.

- **Return 404 (not 403) for cross-user scoped resource access.** When a user accesses another user's preference, calendar connection, or any user-scoped entity, return 404. 403 leaks existence. Test both: own resource → 200, other user's resource → 404. Apply to all CRUD endpoints.

- **Narrow `except` clauses in Celery tasks — never swallow `celery.exceptions.Retry`.** Broad `except Exception:` handlers will swallow `celery.exceptions.Retry`, silently converting retryable errors into permanent failures. Always use narrow exception types or explicitly re-raise `Retry`. Add a test `test_retry_exception_propagates` for any task that has a retry path inside a try/except.

- **ECDSA / HMAC webhook signature validation is mandatory for all inbound webhooks.** Validate raw `request.body()` bytes before JSON parsing. Use provider verification library where available (SendGrid: `EventWebhook.verify_signature`). Fail-closed on empty/missing key. Three required tests: valid → 200, invalid → 401/403, empty-key → 401/403. This is the same `hmac.compare_digest()` rule from E04 extended to ECDSA signatures.

- **Fernet encryption canonical module for third-party OAuth tokens.** All OAuth2 refresh/access token storage must use the `notification/core/token_crypto.py` pattern: Fernet roundtrip test (stored bytes ≠ plaintext, decrypt(encrypt(t)) == t), key sourced from env/Vault, value scrubbed from structlog. Reuse this module — do not re-implement per-service.

- **Stripe usage counter ordering: read → API(idempotency-key) → GETDEL.** This exact sequence prevents both double-billing (idempotency key) and data loss (GETDEL only after confirmed Stripe 200). Two required P0 tests: counter intact on failure, counter cleared on success.

- **Beat schedule constants in the epic spec must be cross-checked against the Beat dict before story creation.** Any story that references a UTC time constant in its ACs must be reconciled against the service's Beat schedule dict (e.g., `BEAT_SCHEDULE` in `beat_schedule.py`). Divergences between spec and implementation produce tested-but-wrong schedules that may not be caught until operational monitoring.

### From Epic 10 Retrospective

- **`bmad-testarch-trace` traceability workflow must run BEFORE the first story begins implementation.** The matrix output is the AC coverage plan — every story knows which AC it satisfies before coding starts. This is the causal factor for Epic 10's first-ever TRACE_GATE PASS. Running it as a retrospective artifact is too late; it must gate the dev queue.

- **Foundational security middleware (e.g., S10.02 proposal-level RBAC) is a blocking gate.** Do not merge features that rely on a security layer until that layer is implemented and verified. UUID-based existence leakage is a high-severity risk. Stories depending on an unimplemented security dependency must not enter `review` status until the security story is `done`.

- **Atomic UPSERT with `ON CONFLICT DO UPDATE WHERE (expires_at < NOW() OR locked_by = EXCLUDED.locked_by)` is the correct serialization point for pessimistic locking.** No application-layer lock loop needed. A single database round-trip either acquires the lock atomically or returns the existing holder's info. The concurrent acquisition test must assert that a second requester with a valid lock gets 423 with lock-holder name and expiry. (Source: S10.03 section locking.)

- **Upsert field partitioning for AI + human decision flows.** When an endpoint stores both AI recommendation and human decision, partition the two operations so `evaluate` preserves `decision`/`decided_*` fields and `decide` preserves `ai_recommendation`/`evaluated_at`. Neither call clobbers the other's fields. Service returns `(response, was_insert)` tuple; router maps to 201/200 accordingly. (Source: S10.10 bid/no-bid decision.)

- **Visibility-aware polling for async AI results.** Any frontend component polling an async AI operation (lessons learned, report generation, AI scorecard) must: (1) bind `document.visibilitychange` to pause polling when tab is hidden and resume on return, (2) use a per-mount attempt counter (not a global variable), (3) cap with a wall-clock bound (e.g., 15s × 20 = 5 min max). Prevents runaway polling accumulating across browser tabs. (Source: S10.16 outcome-lessons poll.)

- **`lessons_learned_status` tracking column pattern for async AI agent results.** Stores `pending` / `in_progress` / `complete` / `failed`. Background task runs with an independent DB session. Poll endpoint reads status and result JSONB. This pattern is reusable for any async AI integration where the caller should not block waiting. (Source: S10.11 bid outcome.)

- **SQL injection structural test required for any story using SQLAlchemy `text()` or raw SQL.** String interpolation on dynamic values in `text()` clauses is an injection risk invisible to ruff, mypy, and ATDD. Add `test_{feature}_uses_bound_params` to the ATDD checklist: assert the function uses `bindparam()` or named parameters, never f-strings on user-supplied values. (Source: S10.09 approval decision engine — caught and fixed in review.)

- **Celery task registration structural test is mandatory.** A function missing `@app.task` imports and unit-tests normally but is never registered in the worker; the production call silently does nothing. Every story introducing a Celery task must include: `assert "service.module.task_name" in celery_app.tasks`. One line; prevents silent no-op deployment. (Source: S10.03 section lock cleanup task.)

- **Upsert endpoints must return 201 on insert and 200 on update — not always 201.** RFC 7231: 201 = resource created; 200 = resource updated. ATDD must assert both: first call → 201, second call with same payload → 200. Service layer returns `(response, was_insert)` to route accordingly. (Source: S10.10 — always returned 201 regardless, caught in review.)

- **Frontend stories covering 3+ distinct user workflows must be split before entering the dev queue.** S10.16 covered approval stepper + bid/no-bid radar chart + outcome recording in one story and generated 37 review action items (10 blocking). The correlation is systematic — this scope pattern always produces review debt. Story LOC > 3,000 is a split signal.

- **Pre-story spec consistency check for FK + CHECK constraint interactions.** When a story's ACs include a CHECK constraint on a nullable column that also has ON DELETE SET NULL on its FK, the constraint is contradictory (FK nullifies the column; CHECK rejects null). Resolve before implementation begins. (Source: S10.04 comments — `resolved_by IS NOT NULL` in CHECK vs FK `ON DELETE SET NULL`.)

- **Exhaustive audit assertions in every backend story.** Verify `action_type`, `entity_type`, `entity_id`, and `after` JSON content for all POST/PATCH/DELETE endpoints. Enforce PII guards (masking bodies in audit logs).

### From Epic 5 Retrospective

- **`SELECT FOR UPDATE SKIP LOCKED` is mandatory for Celery batch queue workers.** Any Celery Beat task that polls a DB table for pending work and processes items in batches must use `SELECT ... WHERE status='pending' FOR UPDATE SKIP LOCKED`. Without it, multiple worker replicas race to claim the same items, producing duplicate processing and state corruption. Items must be marked `processing` before the AI Gateway call and restored to `pending` on failure or `failed` after max retries. See `process_enrichment_queue.py` as the canonical reference.

- **Celery `task_failure` signal handlers must use Redis-backed state, not in-memory dicts.** Module-level Python dicts (`_RUN_ID_REGISTRY: dict[str, str]`) are incompatible with Celery prefork pool: the signal fires in a different OS process — `_pop_run()` returns `None` and audit records are never updated. Use `redis.setex(f"service:run_id:{task_id}", ttl=3600, value=str(run_id))` and `redis.getdel(f"service:run_id:{task_id}")` to correlate signal handlers with audit rows across process boundaries.

- **3-queue Celery isolation prevents scoring/guide work from starving crawl tasks.** For Celery applications mixing task types with different resource costs (quick crawl HTTP calls vs. long per-item AI scoring), use dedicated queues per task type: `pipeline_crawl`, `pipeline_scoring`, `pipeline_guides`. Start workers with `--queues <queue_name>`. Set `worker_prefetch_multiplier=1` + `task_acks_late=True` for fair dispatch. Without isolation, a scoring wave (100+ tasks) starves crawl tasks queued behind them.

- **Dedicated `CollectorRegistry` is mandatory for Prometheus metrics in FastAPI services.** Define `SERVICE_METRICS_REGISTRY = CollectorRegistry()` in a `metrics.py` module; register all metrics (`Histogram`, `Counter`, `Gauge`) to it (not the global default). The `/metrics` endpoint calls `generate_latest(SERVICE_METRICS_REGISTRY)`. This prevents `ValueError: Duplicated timeseries in CollectorRegistry` when pytest reimports modules. See `services/data-pipeline/src/data_pipeline/metrics.py` as canonical.

- **Pagination sentinel termination must use a falsy check.** When consuming a paginated external API where the "end of pages" token may be `None`, `""`, `0`, or a missing key, use `raw_token = response.output.get("next_page_token"); page_token = str(raw_token) if raw_token else None`. A truthy check covers all edge cases; `if raw_token is not None:` does not cover empty-string sentinels.

- **Celery eager mode + testcontainers + respx is the E2E integration test pattern for pipeline services.** For full-chain testing (Beat → task chain → DB writes → Redis stream publish), use `task_always_eager=True`, session-scoped PostgreSQL testcontainer, and `respx.MockRouter`. Runs the complete chain in a single process without a real broker; CI-safe; verifies side effects end-to-end. See `test_e2e_pipeline.py` as canonical.

### From Epic 12 Retrospective

- **All stories with frontend ACs must have a Playwright E2E spec file before being marked `done`.** A RED-phase spec (test.skip) is the minimum deliverable. The spec must be activated (GREEN) before the epic retrospective. Absence of a spec file is the same as absence of implementation evidence for frontend ACs.

- **Tier gate enforcement requires E2E API tests covering all 4 tiers.** For every Professional+ or Enterprise-gated feature, test all tier combinations: Free→403, Starter→403, Professional→200 (or 403 if Enterprise-only), Enterprise→200. Reuse the `tier-gate-enforcement.api.spec.ts` pattern.

- **Analytics charts (Recharts) must be E2E testable via SVG presence assertion.** When implementing Recharts `<BarChart>`, `<LineChart>`, or `<RadialChart>` inside `<QueryGuard>`, the E2E test should assert: (1) `<svg>` element rendered, (2) at least one data bar/line/point present, (3) hover tooltip appears on mouseover. Empty state must show `<EmptyState>` component from `packages/ui`, not a blank chart area.

- **Async Celery tasks require integration-level end-to-end testing.** It is not sufficient to test the API endpoint that enqueues the task. The test must verify: (1) the Celery task was executed, (2) the side effect occurred (S3 object created, email sent via mock, DB row updated), (3) the job status endpoint reflects completion. Use LocalStack for S3 and a SendGrid mock for email in CI.

- **S3 signed URL delivery must be asserted — not assumed.** Any story that stores output in S3 and returns a signed URL (report generation, proposal export, document uploads) must include a test that: (1) verifies the S3 object exists at the expected key, (2) resolves the signed URL via HTTP and receives a 200 with expected content-type, (3) asserts the URL expires within the specified window (24h for reports).

- **VPN-restricted admin services use a single IPAllowlistMiddleware test suite.** Do not duplicate IP restriction tests per-story. Create one comprehensive `test_ip_allowlist.py` covering all routes in the service, executed once. Reference it in each story's traceability entry.

- **RED-phase ATDD specs require an owner and an activation story at creation time.** When `bmad-testarch-atdd` generates test.skip() specs, the story deliverables must include: (1) the spec file, (2) a reference to the story that will activate it (remove test.skip and achieve GREEN). No RED-phase spec should exist at epic close without a named activation story.

- **Non-functional stories (load testing, security audit) must not be marked `done` until evidence files contain actual results.** A template with placeholder dashes is not done. The `load-test-results.md` must contain real p50/p95/p99 numbers. The `security-audit-checklist.md` must have all checkboxes filled with evidence and a sign-off. Entry criteria include confirming the staging environment and tooling are available before the story begins.

- **TEA automation review is a per-story gate, not an epic-level activity.** Update `tea_status` in sprint-status.yaml for each story as it is reviewed. `tea_status: {}` at retrospective time means the quality validation layer was skipped for the entire epic.

### From Epic 6 Retrospective

- **TierGate MUST be a FastAPI per-route `Depends()` on every endpoint returning gated data — never middleware.** Implementing tier enforcement as a per-route dependency makes it impossible to accidentally omit from new endpoints. The strict response-model selection (e.g., `OpportunityFreeResponse` 6-field-only) must happen inside the gate dependency, not in the route handler. A `test_free_response_has_exactly_six_model_fields` unit test prevents field leakage through model evolution. This is the definitive pattern for all revenue-critical tier enforcement.

- **Atomic Lua script (`_LUA` module-level constant) is mandatory for all Redis metering operations.** Never separate GET + INCR across two Redis round-trips — the race window corrupts billing signal. Use `redis.eval()` with a single script: reads current value, returns 429 if ≥ limit, increments and sets EXPIRE atomically. `fakeredis[lua]>=2.21` (lupa package) required for EVAL support in unit tests; testcontainers Redis required for concurrency race tests. See `usage_gate.py::_USAGE_LUA` as canonical.

- **Negative ATDD assertions are mandatory for stories involving streaming protocols (SSE, WebSocket).** For any SSE endpoint, the ATDD checklist must explicitly assert: "Component does NOT use `new EventSource`" (EventSource is GET-only; POST SSE endpoints silently fail). Also assert "Component does NOT use `Axios` for streaming body" (Axios buffers the full response). Use native `fetch` + `ReadableStream` for POST SSE. These structural negative assertions prevent invisible runtime failures that no functional test catches.

- **URL-driven state is mandatory for all filter/sort/pagination UI.** Filter sidebar, sort controls, and pagination must use `useSearchParams()` + `router.replace()` — never `useState`. Add stale-closure prevention via `searchParamsRef = useRef(searchParams)` updated on every render. This enables shareable URLs, survives client-side navigation, and eliminates the lost-state-on-back-button problem.

- **Headless orchestration components (`'use client'` + `return null`) must receive store accessors as parameters.** Components that register Axios interceptors or global effects should render `return null` and mount exactly once in layout. The accessor function must be passed as a parameter (e.g., `show` from `useUpgradePromptStore`) rather than called with `getState()` inside — prevents circular dependency between `packages/ui` and `apps/client`. Always call `return Promise.reject(error)` in error handlers to preserve TanStack Query error states.

- **Dual-session service functions are required for any service spanning the pipeline/client schema boundary.** Functions reading from `pipeline.*` use `get_pipeline_readonly_session` (separate `MetaData(schema="pipeline")` instance); functions writing to `client.*` use `get_db_session`. Never reuse the same session for both schemas. This prevents cross-schema FK violations and enforces the architectural boundary.

- **UsageGate check MUST execute before `StreamingResponse` is created.** Once `StreamingResponse` is returned to FastAPI, HTTP headers (200 + `Content-Type: text/event-stream`) are committed. A 429 raised inside the generator after headers are sent cannot change the status code — the client receives a success stream followed by an error event. Always check and increment quota atomically, then return `StreamingResponse(...)`.


### From Epic 7 Retrospective

- **Security hardening story injection is the correct NFR FAIL response.** When NFR assessment returns FAIL on security/maintainability, inject a P0 blocking hardening story (reference: S07.17). ACs must derive directly from NFR finding items; the story resolves within the same sprint. When security mitigations are designed-in from story ACs rather than retrofitted, NFR assessment produces PASS (with CONCERNS) instead of the FAIL→injection cycle.

- **Content-hash optimistic locking is the standard for collaborative document editing.** Use a `content_hash` column (SHA-256), client sends the current hash in the request body, `SELECT FOR UPDATE` + hash comparison inside a transaction, return `409 Conflict` with a structured body on mismatch, and display a conflict resolution dialog on the frontend. Both backend (content save API) and frontend (conflict dialog) stories are required to close R-001-class data loss risks.

- **`ThreadPoolExecutor` is mandatory for CPU-bound library calls in FastAPI.** Any endpoint calling WeasyPrint, python-docx, Pillow, lxml, or any CPU-bound library MUST use `await loop.run_in_executor(executor, fn, *args)`. Event loop blocking is invisible in unit tests and only manifests under load. ATDD checklists for export/render stories must assert "function uses `run_in_executor`".

- **Reset-stuck background task is required for all long-running async state machines.** Any resource that transitions through an in-progress status (generation, scanning, export, crawler run) MUST have a Celery Beat cleanup task marking stale entries `FAILED` after a configurable TTL. Reference: S07.05's `reset_stuck_proposals_task`. Add as a required AC to all stories introducing a `status` field with an in-progress variant.

- **TEA test review must gate `review → done` story transitions; minimum score ≥ 80/100.** TEA review catches backend/frontend contract schema drift and spec-vs-implementation constant contradictions invisible to ATDD. S07.11 TEA review (95/100) caught `ProposalResponse` missing `current_version_number`/`generation_status` and a 1280px vs 1024px breakpoint contradiction that unit tests did not surface.

- **Vitest source-inspection ATDD scales to complex multi-panel frontend components.** Regex and import assertions without runtime rendering are stable in CI and enable RED-phase tests before components exist. Reserve RTL/JSDOM for user-interaction flows; S07.15 (87 tests, all GREEN) is the canonical reference.

### From Epic 8 Retrospective

- **Clean billing service boundary is the reference architecture for payment-adjacent services.** Separate modules: `billing_service.py` (Stripe customer/subscription), `webhook_service.py` (idempotent event processing), `vies_service.py` (VAT/VIES with fallback), `tier_gate.py` (FastAPI `Depends()` gating), `tier_cache.py` (Redis + DB fallback), `api/v1/billing.py` (thin router). No cross-service DB joins; all events via Redis Streams; 100% isolated in `client-api/services/billing/`.

- **External validation services on the registration critical path must fail-open to `pending`.** VIES (or any external validation service) returning 503/timeout MUST NOT block registration. Pattern: timeout/503 → `status: pending`; return 201 to caller; retry on next billing event; apply reverse-charge only on `status: valid`. Three integration tests required: valid → synced, invalid → 422, service down → pending+200.

- **External account provisioning (Stripe, CRM) must run as a `BackgroundTask` after the resource creation endpoint returns 201.** The provisioned ID is nullable until populated. All billing endpoints return 422 with a structured error on missing ID. This prevents external service outages from blocking user registration.

- **Webhook dedup table with `event_id` unique constraint is the mandatory idempotency mechanism for all payment webhooks.** Pattern: `INSERT INTO webhook_events(stripe_event_id)` inside the processing transaction; unique-constraint violation → return 200 (already processed). Not in-memory state or Redis TTL. Reference: S08.04 `webhook_events` table.

- **Event-driven cache invalidation must DELETE the cached key, never SET it to an assumed new value.** `DELETE` forces the next request to read the authoritative DB value. `SET` requires knowing the new value at consumer time — creating a read-before-write race. Reference: S08.14 `tier_cache.delete(company_id)` pattern.

- **`asyncio.to_thread()` is mandatory for all synchronous I/O SDK calls inside `async def` FastAPI handlers.** Stripe Python SDK, VIES SOAP clients, and any synchronous HTTP/DB library MUST be wrapped with `await asyncio.to_thread(sync_fn, *args)`. This extends the E07 ThreadPoolExecutor rule (CPU-bound) to I/O-bound synchronous SDKs.

---

## Anti-Patterns (Don't Do This)

### From Epic 1 Retrospective

- **Don't skip TEA automation for "simple" stories.** Stories 1.8–1.10 (CI, Helm, Terraform) were missing formal TEA automation summaries despite having tests. Every story needs a TEA review cycle.
- **Don't defer the same issue across multiple reviews without consolidating.** Docker root user was flagged in Stories 1.1 and 1.2 reviews separately. Consolidate into one tracked item.
- **Don't leave sprint-status.yaml inconsistent.** Update epic status to `done` when all stories complete, before the retrospective.
- **Don't use `ignore_missing_imports = true` globally in mypy.** Narrow to specific third-party modules to catch misspelled imports.
- **Don't bind Docker Compose ports to 0.0.0.0.** Use `127.0.0.1:<port>:<port>` to prevent network exposure.
- **Don't use `:latest` tags on infrastructure Docker images.** Pin specific versions for reproducibility.

### From Epic 2 Retrospective

- **Don't call `bcrypt.hashpw` directly in async context.** This blocks the event loop for 200–400ms, starving all concurrent requests. Always use `run_in_executor`.
- **Don't forget `User.is_active` in authentication queries.** Every query that grants a JWT (login, refresh, accept_invite) must verify the user is active.
- **Don't let Redis errors propagate as 500 from the rate limiter.** Fail-open: log the error, skip the rate limit check, continue processing. Rate limiting is a hardening layer, not a hard dependency.
- **Don't accept unbounded password input.** All password fields must have `max_length=128` (or similar) to prevent bcrypt HashDoS.
- **Don't use `HTTPException` for 429 rate-limit responses.** Use `JSONResponse` directly to produce the standard `{"error": ..., "message": ..., "details": null, "correlation_id": "..."}` envelope.
- **Don't duplicate deferred work entries across multiple code reviews.** When a cross-cutting issue (e.g., blocking bcrypt, ORM type annotations) appears in multiple stories, consolidate to one entry in deferred-work.md with references to all source stories.
- **Don't allow `isinstance(data, dict)` branch logic in response assertions.** Commit to the declared schema. Use `assert "versions" in data` — a wrong key name in production should fail the test immediately with a clear error, not silently return an empty list.
- **Don't leave `_SKIP_REASON` constants or `@pytest.mark.skip` decorators in tests after GREEN phase.** Clean up RED-phase scaffolding before marking a story done.
- **Don't defer NFR HIGH-priority items from one epic to the next without re-surfacing them.** The NFR report must include a "Carry-Forward from Previous Epic" section. Items not addressed become automatic CONCERNS in the new epic's assessment.

### From Epic 3 Retrospective

- **Don't use `user?.companyId ?? "some-stub-id"` in any component.** Stub fallback IDs silently send invalid data to real APIs at integration time. Always guard: `if (!user?.companyId) { showError(...); return; }`.
- **Don't hardcode display strings in any UI component.** Every user-visible string — labels, placeholders, error messages, empty states, breadcrumbs — must use `useTranslations()` with a key from `bg.json`/`en.json`. Hardcoded strings break BG locale.
- **Don't mark a frontend story done with incomplete TEA.** TEA automation is a hard gate, same as for backend stories. `tea_status: in-progress` blocks the epic retrospective.
- **Don't skip the redirect count assertion when modifying locale routing.** Silent redirect loops are invisible in manual testing and only surface under load or with specific browser cookie states.
- **Don't use ad-hoc conditional renders for loading/error/empty states.** This creates inconsistent UX and bypasses the unified `<QueryGuard>` + skeleton/error boundary/empty state system. Every data-fetching component uses `<QueryGuard>`.
- **Don't share Zustand persist keys between client and admin apps.** `eusolicit-auth-store` must be `eusolicit-client-auth-store` and `eusolicit-admin-auth-store`. Same-origin deployment (behind a reverse proxy) would cause the admin app to read client user tokens.
- **Don't fire multiple `POST /auth/refresh` calls concurrently.** E02's token family revocation logs the user out on second refresh. The `apiClient` singleton refresh-lock is the protection. Never bypass or replace it with a simpler retry pattern.
- **Don't use `"turbo": "latest"` in `frontend/package.json`.** Each fresh `pnpm install` may pull a different Turborepo major version (e.g., v2 → v3), silently breaking the build pipeline. Always pin to a specific minor range: `"turbo": "^2.9.5"` (or the current installed version). Check with `pnpm list turbo` and update the pin after intentional upgrades.

### From Epic 4 Retrospective

- **Don't use naive string equality (`==`) for HMAC signature comparison.** `header_sig == computed_sig` is vulnerable to timing attacks — an attacker can determine the correct signature one byte at a time by measuring response time differences. Always use `hmac.compare_digest()`.

- **Don't skip ATDD checklists for backend epics.** Epic 4 produced zero `atdd-checklist-4-*.md` files across 10 stories. The ATDD cycle (RED tests before implementation) is the discipline that proves test-driven development — skipping it makes the test design a post-hoc audit rather than a development gate. ATDD checklist scaffold must be created before story development starts, not after.

- **Don't let circuit breaker state live only in memory in multi-replica environments.** In-memory `asyncio.Lock`-guarded state is per-process. When a service scales to N replicas, each instance has independent circuit state: one instance may reject while another continues to hammer a failing agent. Document the limitation at implementation time; do not defer the Redis migration decision beyond the first auto-scaling event.

- **Don't implement SSE proxy without a wall-clock latency benchmark.** Functional tests (events forwarded in order) do not verify the < 100ms per-event latency requirement. SSE latency is a user-experience guarantee — verify it with a controlled mock SSE source with known send timestamps and `time.monotonic()` comparison before marking the story done.

- **Don't close an epic without an NFR assessment when the epic introduces stateful resilience, outbound HTTP clients, or streaming.** Epic 4 had no NFR report. The circuit breaker per-instance gap (E04-R-005), SSE latency unverified (E04-R-002 residual), and Redis publish reliability (E04-R-003 monitoring backlog) are NFR concerns that compound over time if not formally assessed.

- **Don't defer the same security hardening items across more than two consecutive epics.** Items deferred from E01 that remained open through E04 (Prometheus /metrics, Dockerfile USER, Dependabot, bcrypt executor, User.is_active, Redis fail-open, JWT audience/issuer) are now 4 epics overdue. Each additional deferral increases integration risk. Cap deferral at 2 consecutive epics — after that, inject as a mandatory story in the next sprint.

### From Epic 9 Retrospective

- **Don't leave a story file at its pre-dev status after the dev phase completes (RC2 — 2nd recurrence).** Story 9.3's file retained `Status: ready-for-dev` with all task checkboxes unchecked despite 12 tests being implemented and passing per the ATDD checklist. The story file is the source of truth for the traceability matrix and gate decisions — a wrong status propagates to TRACE_GATE: CONCERNS. Update the status field, check all task boxes, and add a `## Dev Agent Record` section before emitting HALT. This is the same Root Cause 2 from the Story 5-5 retrospective. Layer B (`2b-dev-story-verify` phase) must be prioritised.

- **Don't defer frontend component tests without naming an activation story.** Stories 9.12 and 9.13 deferred Vitest + Testing Library component tests to "TEA phase" with no sprint or story designated to activate them. Per the E12 retrospective rule, every deferred test category must specify (1) the test file stub, (2) the story that will activate it, (3) the sprint target. "Deferred to TEA phase" alone is not acceptable.

- **Don't ship a new long-running service without Prometheus metrics.** The Notification Service deployed with ~810 tests and zero observability instrumentation. Any new service that runs periodic tasks, Celery workers, or SLA-bound operations must define its metrics in the same sprint as the service scaffold (S09.01 equivalent). Metrics are not a Sprint N+1 concern.

- **Don't leave `epic-N: in-progress` in sprint-status.yaml when all stories are `done` (4th recurrence).** `epic-9: in-progress` was set despite all 14 stories being `done`. Add a CI quality gate that enforces this — `quality-gates.yml` must fail on this condition. This has now occurred in E07, E09, E11, E12.

### From Epic 10 Retrospective

- **Don't skip epic-level test design.** Epic 10 skipped the `test-design-epic-10.md` pipeline, leaving NFR verification undefined. NFR strategy must be established during the planning phase.

- **Don't defer security middleware as a non-blocking item.** Delayed implementation of foundational security layers (e.g., S10.02) compromises the integrity of all downstream features. Security middleware is always a P0 blocking dependency.

- **Don't allow systematic TEA review under-execution.** 8 consecutive epics with 0 TEA scores indicates a failed process gate. Orchestrator must block `review -> done` transitions on TEA score presence.

- **Don't use f-string interpolation in SQLAlchemy `text()` clauses for dynamic values.** SQL injection risk invisible to ruff, mypy, and ATDD. Found in S10.09 approval decision engine: `WHERE stage_id = '{stage_id}'` instead of `bindparam("stage_id")`. Add structural test asserting bound parameters are used.

- **Don't ship Celery tasks without verifying worker registration.** A function missing `@app.task` passes all unit tests (the function exists) but is never registered in the worker process. Found in S10.03: cleanup task was callable but never registered. Add `assert task_name in celery_app.tasks` to every story's ATDD checklist.

- **Don't bundle 3+ distinct user workflows in a single frontend story.** S10.16 (approval stepper + bid/no-bid radar chart + outcome recording) generated 37 review action items (10 blocking), project record. Split at story creation time — not after review discovers the scope.

- **Don't return HTTP 201 unconditionally from upsert endpoints.** RFC 7231 requires 200 for updates. S10.10 always returned 201; found in review. Service must return `(response, was_insert)`; router maps to 201/200. ATDD must test both codes.

- **Don't let injected remediation stories sit at `ready-for-dev` across multiple epics.** `inj-01` (Dependabot), `inj-02` (k6), `inj-03` (TEA backlog) have been `ready-for-dev` for 8 consecutive epics. Injected stories with CRITICAL/HIGH severity must be P0 blockers on the next epic close gate, not optional backlog items.

- **Don't allow contradictory spec constraints to reach implementation.** S10.04 had a CHECK constraint requiring `resolved_by IS NOT NULL` when `resolved = TRUE`, combined with an FK `ON DELETE SET NULL` on `resolved_by`. The constraint is impossible to satisfy after user deletion. Resolve FK + CHECK contradictions in story creation, not mid-implementation.

### From Epic 5 Retrospective

- **Don't use a module-level dict (`_RUN_ID_REGISTRY`) to correlate Celery task IDs with DB record IDs in signal handlers.** Celery prefork pool fires `task_failure` signals in a different OS process than the task body. The in-memory dict is not shared across processes — `_pop_run(task_id)` returns `None`, the audit record (`crawler_runs.status`) is never marked `failed`, and orphaned `running`/`retrying` rows accumulate silently. Tests pass because they use eager mode (single process). Always use Redis SETEX/GETDEL for signal-handler ↔ task-body state correlation.

- **Don't use separate DB sessions for audit-record creation, data upsert, and audit-record status update within the same Celery task.** Three separate `get_sync_session()` contexts create a partial-failure window: if the process crashes between the second and third session, data is persisted but the audit record remains in `running` forever with no automatic recovery. Minimise session boundaries; add a startup recovery sweep task to reclaim stuck records.

- **Don't skip ATDD checklists for backend epics — 3rd recurrence (E04, E05, and prior).** E05 produced zero `atdd-checklist-5-*.md` files despite E04's critical commitment. The cultural norm is insufficient — the orchestrator must mechanically block the `2-dev-story` phase from starting without a checklist file present. ATDD is a development gate, not a post-hoc audit.

- **Don't leave a story file at its pre-dev/pre-review status after the story completes (RC2 — 3rd recurrence).** Story 5.5 file retained `Status: ready-for-review` while sprint-status.yaml shows `5-5: done`. Downstream traceability tools treat the story file as the source of truth. A wrong status propagates to TRACE_GATE: CONCERNS and misleads the orchestrator. Update `Status:`, check all task boxes, and add the `## Dev Agent Record` section before emitting HALT. The `2b-dev-story-verify` orchestrator phase is the mechanical fix.

- **Don't design circuit breakers with a broad `except Exception` catch that counts 4xx responses as failures.** `circuit_breaker.py` with a generic `except Exception` handler that increments the failure counter for all exception types will prematurely open the circuit on repeated 400 responses (e.g., misconfigured normalization payload). Distinguish retriable errors (5xx, transport, timeout) from non-retriable client errors (4xx) at the circuit-breaker level. Only the former should increment the failure count.

- **Don't read env vars for batch-size limits without guarding against 0 or negative values.** `batch_size = int(os.environ.get("ENRICHMENT_BATCH_SIZE", "20"))` with no guard silently processes nothing when `ENRICHMENT_BATCH_SIZE=0`. Always apply `batch_size = max(1, batch_size)`.

### From Epic 12 Retrospective

- **Don't mark a story `done` if it has frontend ACs and no E2E spec file.** Absence of a spec file means the AC is unverified at the browser level. A backend API test for a frontend AC is not sufficient — it proves the API works, not that the UI renders correctly.
- **Don't treat a non-functional story (load test, security audit) as done when its output files contain only templates.** Placeholder dashes in load-test-results.md and unchecked boxes in security-audit-checklist.md are not evidence. These stories have hard output artifact requirements.
- **Don't generate Celery tasks without integration test coverage of their execution.** Testing the endpoint that enqueues a task is not the same as testing the task. The side effects (S3 upload, DB write, SendGrid call) must be verified end-to-end.
- **Don't leave `test.skip()` decorators in E2E specs at epic close without a named activation story.** RED-phase specs are a deliverable; their activation plan is a separate required deliverable.
- **Don't skip TEA reviews because a story is "fully covered" or "simple".** Test quality and isolation can only be validated by a formal TEA pass. High test counts do not guarantee test quality.
- **Don't leave `epic-N: in-progress` in sprint-status.yaml when all stories are `done`.** This is the 4th occurrence of this anti-pattern. It will be enforced by CI going forward.
- **Don't implement analytics queries without `WHERE company_id = :cid` scoping on every materialized view query.** Cross-tenant data leakage in analytics is an R12.1 (Score 6) risk. Every analytics service method must include a cross-tenant negative test in its test suite.
- **Don't use `REFRESH MATERIALIZED VIEW` without `CONCURRENTLY` on production tables.** Non-concurrent refresh takes an exclusive lock that blocks all reads during refresh. Always add a `UNIQUE` index on materialized view columns used for filtering before enabling `CONCURRENTLY`.

### From Epic 6 Retrospective

- **Don't implement SSE endpoints without the four generator lifecycle checks.** The following four bugs will reach production if not explicitly guarded: (1) `asyncio.shield(semaphore.acquire())` causes permanent permit leaks on timeout — use `await asyncio.wait_for(semaphore.acquire(), timeout=X)` directly; (2) usage/quota check not done before `StreamingResponse` creation — 429 inside generator cannot change already-committed 200 headers; (3) `asynccontextmanager` stream generator not closing inner async generator in `finally` — httpx resources leak on consumer cancellation; (4) no terminal event emitted when upstream stream ends without `done`/`error` — client is left with orphaned connection. Add all four to the SSE story spec checklist.

- **Don't deploy frontend components that consume upstream query hooks without declaring the exact query key.** A query key mismatch (e.g., `["opportunity-detail", id]` vs. the upstream `["opportunity", id]`) causes `invalidateQueries` to silently fail — the cache is never invalidated, stale data is displayed, and no error is raised. Story specs for frontend components consuming upstream hooks MUST include a section "Required query keys from upstream stories" with the exact key array copied from the source story.

- **Don't mark a story `done` unless ALL three completion gates pass.** The three required gates are: (1) Senior Dev Review APPROVED (not just "patches applied"), (2) ALL ATDD tests GREEN (not "tests written but skipped"), (3) i18n parity check passing (`pnpm check:i18n` shows no missing keys). A story meeting only two of three gates is `in-review`, not `done`. This applies equally to backend and frontend stories.

- **Don't generate traceability matrices before the implementation spec (story file) is locked.** A traceability matrix generated concurrently with story authoring can diverge from the spec (e.g., 100MB vs 50MB file size limit). The correct sequence is: lock story spec → generate ATDD checklist → implement → generate traceability matrix from the ATDD checklist. ATDD checklists are the source of truth; traceability derives from them.

- **Don't specify Redis-gated features without an explicit fail-open requirement.** If Redis is unavailable, a Redis-gated feature (UsageGate, rate limiter) must allow the request through, not fail it. Template: "If Redis is unavailable, allow the request, log `structlog.warning('redis_gate_degraded')`, and set `X-{Feature}-Remaining: -1`." This rule already applies to the rate limiter (E02); extend it to every new Redis-gated feature.

- **Don't carry the same infrastructure blocker past a 4th epic.** Dependabot, Prometheus /metrics, and k6 baseline have each been deferred for 4+ consecutive epics. The rule: any infrastructure item deferred twice becomes a mandatory story in the next sprint. A 4th deferral is a process failure — escalate to the operator and block the relevant milestone gate.


### From Epic 7 Retrospective

- **Don't maintain frontend TypeScript types manually — generate from backend OpenAPI spec.** Manually maintained frontend type files diverge from backend Pydantic schemas. `ProposalResponse` was missing `current_version_number` and `generation_status`, only caught in TEA review. Frontend types MUST be generated via codegen (`openapi-typescript`). Manual type duplication is prohibited for any backend-derived response type.

- **Don't omit numeric constants from story acceptance criteria.** Breakpoint thresholds (e.g., 1024px vs 1280px), file size limits, timeouts, and retry counts must be listed in an "Implementation constants" subsection of each story AC. Undeclared constants cause silent implementation divergence only caught in TEA review.

- **Don't mark a story `done` if any P0 AC has 0% confirmed-passing (GREEN) tests.** Stories with P0 ACs that have only RED/written tests cannot be `done`. A story is `in-review` until ≥1 GREEN test per P0 AC is confirmed. This is the per-story gate that prevents TRACE_GATE failures where test existence is 100% but quality assurance is zero.

- **Don't state ATDD test count totals that diverge from level-by-level breakdowns.** A header total that doesn't match the sum of individual levels creates misleading coverage metrics. Add an auto-computed total to the ATDD checklist template; CI lint should validate the summary total matches the breakdown.

### From Epic 8 Retrospective

- **Don't treat retrospective [ACTION] items as advisory.** E07 retro designated k6 baseline, Dependabot, and S07.04 R-001 optimistic locking GREEN verification as Epic 8 hard deliverables. None were completed. Every `[ACTION]` item with `SEVERITY: critical` must have an Orchestrator verification task checked at the next retrospective before new items are accepted. Unresolved critical carry-forwards must block epic kickoff.

- **Don't classify E2E specs with all tests wrapped in `test.skip()` as PARTIAL coverage — they are NONE coverage.** A spec file where ≥80% of tests are `test.skip` consumes engineering time without providing quality assurance. ATDD checklists must include E2E test activation as a blocking sub-task before a story is marked `done`.

- **Don't deploy Stripe or any outbound payment SDK calls without circuit-breaker protection.** The E04 two-layer resilience pattern (`circuit_breaker(retry(http_factory))`) is the standard for ALL outbound HTTP. Simple `try/except` logging does not prevent cascading failures on a degraded external endpoint. All outbound payment API calls must adopt this pattern.

- **Don't deploy revenue-critical paths without Prometheus metrics.** Billing webhook processing latency, usage sync drift, Stripe API error rate, per-tier subscription counts, and trial-to-paid conversion rate are required operational metrics. A billing failure invisible until a user complains is a platform reliability failure.

### From Epic 16 Retrospective

- **Don't deliver a new service's first dev pass without verifying the service starts.** (AP16-01, SEVERITY: critical) The first dev pass on a new FastAPI service must pass a minimal "service actually starts" check before being labelled review-ready: `uvicorn <service>.main:app --workers 1` exits cleanly under test settings, at least one real (non-skipped) test passes, and the Dev Agent Record contains a real pytest summary line. A skeleton with `@pytest.mark.skip` on all tests and placeholder handler dicts is NEVER review-ready. Apply the PB-ZEROOUT-007 self-check to every new service story before emitting HALT.

- **Don't assign a port to a new service without reading all existing host-port mappings in `docker-compose.yml`.** (AP16-02, SEVERITY: high) Port collisions require additional remediation passes with no business value. Before assigning a port number, read `docker-compose.yml` and list every `ports: "<host>:<container>"` entry, then cross-reference CLAUDE.md's service port table. The story AC for any new service must include the specific port number. `integrations-api` correctly settled on port 8007 after two incorrect assignments (8002, 8006).

- **Don't leave a story file `Status:` header at `review` after the final code review Approve. (5th recurrence: E09, E12, E13, E15, E16).** (AP16-03, SEVERITY: high) The story file header is the source of truth for NFR assessment, traceability, and retro tools. A stale `review` status causes downstream tools to report false blocking findings. The `2b-dev-story-verify` orchestrator phase must verify story file `Status: done` matches sprint-status `done` value before closing any story.

- **Don't treat `@resilience_pattern` as the two-layer pattern until it includes a circuit-breaker outer layer.** (AP16-04, SEVERITY: high) `eusolicit_common.resilience.resilience_pattern` provides retry-only. AC-4 #6 / Rule 47 require `circuit_breaker(outer) → retry(inner)`. Any story using `@resilience_pattern` for outbound HTTP to an external service (Slack, Teams, CRM, Stripe) is not compliant with Rule 47 until the circuit-breaker layer is implemented. The structural test must validate circuit-breaker semantics, not just decorator presence. Target: implement in `eusolicit_common.resilience` before E17 Story 17-0.

- **Don't create a story for a single-story epic without explicitly verifying all epic-spec ACs are covered.** (AP16-05, SEVERITY: high) E16's S16.0 was intended to be the only story in the epic but its ACs cover only the backend. The epic spec required throttling (Redis SETNX), i18n (BG/EN notification content), and frontend configuration UI — none were in story ACs. When story ACs don't cover all epic ACs, the missing ACs must be explicitly deferred to a named follow-up story in the story spec (not left silent). A story that omits epic-spec ACs without documentation will produce incomplete implementation.

- **Don't accept a traceability matrix generated at RED phase as the definitive quality gate for an epic.** (AP16-06, SEVERITY: medium) When the traceability matrix is generated during ATDD checklist creation (before implementation), it shows 0% coverage and GATE: FAIL by design. This stale artifact misleads retro readers and does not capture the actual test state after implementation. The traceability matrix must be regenerated after the last remediation pass reaches Approve status. Stale matrices create false urgency and hide real scope gaps.

- **Don't start E17 without resolving the `sync_logs` vs `processed_alerts` table schema conflict.** (AP16-07, SEVERITY: medium) The E16 epic spec designed `integrations.sync_logs` as the shared delivery tracking table for both E16 (Slack/Teams) and E17 (CRM). S16.0 created `integrations.processed_alerts` instead (different columns). E17's CRM sync engine needs `provider`, `direction`, `entity_type`, `entity_id`, `status`, `error`, `occurred_at` columns. Either extend `processed_alerts` or create `sync_logs` — decide before S17-0 is created.

### From Epic 16 Retrospective — Patterns

- **AST-level structural tests are the standard for security property verification.** (Source: S16.0 `test_static_security.py`) The pattern: (1) walk the service AST to confirm `@resilience_pattern` is applied to outbound HTTP functions, (2) confirm `hmac.compare_digest` is absent from outbound webhook dispatch (correct — no inbound validation needed for outbound-only services), (3) walk all `logger.*` call sites and assert no sensitive values (`webhook_url`, `decrypted_url`, `token`) appear as named kwargs. These tests are refactor-resistant and run in milliseconds. All new integration services must include an equivalent `test_static_security.py`.

- **Shared crypto helper is the only Fernet implementation.** `eusolicit_common.crypto.FernetCrypto` is the authoritative Fernet module. Any service storing secrets at rest (webhook URLs, OAuth tokens, API keys) imports this helper. Decrypt immediately before use; never store in a variable beyond the outbound call scope. The legacy `notification/core/token_crypto.py` should be migrated onto the shared helper in a future story.

- **Consumer group naming `cg:{service-name}:{stream-name}` is the canonical pattern for multi-consumer streams.** When two or more services subscribe to the same Redis Stream (e.g., `alerts`), each service must use its own consumer group. The naming convention `cg:integrations-api:alerts` (distinct from `cg:notification:alerts`) ensures both services receive every event independently. Always include the consumer group name as a named constant in the story Dev Notes.

- **`--fail-on-skipped` CI gate backed by a `pytest_sessionfinish` hook is mandatory for new services.** The flag alone may not be available in all pytest versions. The service `conftest.py` must implement a `pytest_addoption` + `pytest_sessionfinish` plugin that counts skipped tests and fails the session if any are detected. This ensures the CI gate fails hard on leftover RED-phase `@pytest.mark.skip` decorators.

- **Idempotency unique constraint for fan-out consumers must include the delivery-target identifier.** When a consumer dispatches one event to N recipients, the idempotency table unique constraint must include the per-recipient key (e.g., `webhook_id`), not just the event identifier + category. `(workspace_id, alert_id, integration_type)` incorrectly merges all webhooks of the same type — use `(workspace_id, alert_id, webhook_id)`. This ensures each delivery destination tracks independently and no webhooks are silently dropped on the second+ dispatch for the same event.

---

## Known Technical Debt

Key items from deferred-work.md (126+ items across E01–E12 — see full list there):

| Category | Key Items | Priority | Source Epic |
|----------|-----------|----------|------------|
| **Security** | No `User.is_active` check in login+refresh; no JWT audience/issuer verification; bcrypt password truncation; no `max_length` on password fields; concurrent token rotation race; Dockerfiles run as root; ports on 0.0.0.0; no Dependabot; no OAuth CSRF state validation; AuthGuard no hydration timeout; Zustand persist keys not namespaced per app | **HIGH** | E01, E02, E03 |
| **AI Gateway** | Circuit breaker per-instance state (E04-R-005 — must migrate to Redis before multi-replica); SSE latency < 100ms unverified (E04-P1-009 has no wall-clock assertion); `agents.yaml` 29-entry count only verified by nightly P3 test; config negative-path test missing (E04-P2-NEW-001); per-agent concurrency limit observability incomplete | **MEDIUM** | E04 |
| **Concurrency** | Token rotation race (no SELECT FOR UPDATE); accept_invite TOCTOU; last-admin TOCTOU; rate limiter INCR+EXPIRE non-atomicity; non-atomic DLQ move; multi-tab wizard sync | **HIGH** | E01, E02, E03 |
| **Performance** | Blocking bcrypt on async event loop; load test p95 results NOT DOCUMENTED (E12 S12.17 empty template); no k6/Lighthouse performance baseline confirmed across any endpoint; analytics MV query latency under production load unverified | **HIGH** | E01–E12 |
| **Observability** | No Prometheus /metrics endpoint; no frontend error reporting (Sentry); no structured logging in auth_service; no `x-request-id` correlation header; no Grafana/Loki stack; no pnpm audit in CI; pip-audit not run against services (E12 OWASP A06 unchecked) | **HIGH** | E01, E02, E03 |
| **Data Integrity** | No UNIQUE on entity_permissions; no UNIQUE on espd_profiles(company_id, version); email case not normalized; `stub-company-id` fallback in wizard | MEDIUM | E02, E03 |
| **Reliability** | Redis outage blocks all logins; RSA key loss invalidates all JWTs; no circuit breaker in apiClient for repeated 500/503; Celery Beat schedule drift/miss undetected (no integration-level firing test) | **HIGH** | E02, E03, E12 |
| **Test Coverage** | Frontend E2E absent for S12.03, S12.04–S12.08 UI, S12.14, S12.18 (38 ACs uncovered); S12.09 PDF/DOCX + S3 generation untested; S12.10 Celery Beat trigger + SendGrid untested; 12 RED E2E specs never activated; OWASP checklist unsigned | **HIGH** | E12 |
| **Notification Service** | Story 9.3 story-file status inconsistency (file=`ready-for-dev`, sprint-status=`done`, ATDD checklist=12 tests); epic-9 status not closed; 60s immediate-alert SLA unmeasured on staging; component tests S09.12/S09.13 deferred with no activation story; no Prometheus metrics or Grafana dashboard; calendar sync partial-failure test missing (E09-R-007); no E06↔E09 contract test; iCal token log redaction unverified | **HIGH/MEDIUM** | E09 |
| **Build/Tooling** | `turbo: latest` in package.json; `@typescript-eslint` v6 EOL; `pnpm engines` mismatch; `@/*` path alias points to non-existent `src/`; UI package no build step | MEDIUM | E03 |
| **Integration Readiness** | OAuth callback AbortController missing; OAuth error params not surfaced; loginUser response validation; `stub-company-id` removal | **HIGH** | E03 |
| **Code Quality** | `Mapped[sa.UUID]`/`Mapped[sa.DateTime]` annotations project-wide; global mypy ignore_missing_imports; fragile CI test filter | MEDIUM | E01, E02 |

---

## Project Structure Reference

```
eusolicit-app/
├── services/{client-api,admin-api,data-pipeline,ai-gateway,notification}/
│   ├── src/<service_name>/
│   ├── tests/
│   ├── pyproject.toml
│   ├── Dockerfile
│   └── alembic/
├── packages/{eusolicit-common,eusolicit-models,eusolicit-kraftdata,eusolicit-test-utils}/
├── frontend/                    # Next.js 14+
├── infra/
│   ├── helm/eusolicit-service/  # Shared base chart
│   ├── helm/values/             # Per-service values
│   ├── terraform/               # Scaffold with modules
│   └── postgres/init/           # Schema + role init SQL
├── .github/workflows/           # ci.yml, quality-gates.yml, test.yml
├── docker-compose.yml
├── Makefile
└── pyproject.toml               # Root workspace config
```

---

---

## Auth & Identity Layer Reference

_Quick reference for agents working on epics that depend on E02 (all epics with authenticated endpoints)._

### JWT Claims

```python
# Access token claims (RS256, 15-min expiry)
{
    "sub": "<user_uuid>",
    "company_id": "<company_uuid>",
    "role": "admin|bid_manager|contributor|reviewer|read_only",
    "exp": <unix_timestamp>,
    "iat": <unix_timestamp>,
    "jti": "<token_uuid>"
}
```

### Role Hierarchy

```
admin (5) > bid_manager (4) > contributor (3) > reviewer (2) > read_only (1)
```

### Permission Ceilings by Role

| Role | Max Permission |
|------|----------------|
| admin | manage (bypass entity-level check for own company) |
| bid_manager | manage (bypass entity-level check for own company) |
| contributor | write |
| reviewer | read |
| read_only | read |

### Key API Patterns

- **Protected endpoint:** Depends on `CurrentUser = Depends(require_auth)` — returns `CurrentUser(user_id, company_id, role)`
- **Role-gated endpoint:** `Depends(require_role("bid_manager"))` — enforces minimum role level
- **RBAC-gated endpoint:** `Depends(require_permission("write", entity_type="opportunity"))` — checks entity-level permission with role ceiling
- **Company-scoped query:** Always filter on `company_id` matching `current_user.company_id`. Never trust user-supplied `company_id` without verification.

---

---

## Frontend Structure Reference

```
frontend/
├── apps/
│   ├── client/                    # Next.js 14 App Router — main user app (port 3000)
│   │   ├── app/[locale]/
│   │   │   ├── (auth)/            # Login, register, forgot-password (no AppShell)
│   │   │   ├── (protected)/       # AuthGuard layout — dashboard, tenders, etc.
│   │   │   ├── dev/               # /dev/components, /dev/form-test, /dev/api-test, /dev/toasts
│   │   │   └── layout.tsx         # NextIntlClientProvider + QueryProvider root
│   │   ├── messages/{bg,en}.json  # i18n translation files (6 namespaces)
│   │   ├── i18n.ts                # getRequestConfig — locale detection
│   │   └── middleware.ts          # next-intl locale routing + session cookie guard
│   └── admin/                     # Next.js 14 App Router — admin app (port 3001)
│       └── ...                    # Same structure as client
└── packages/
    ├── ui/                        # Shared component library
    │   └── src/
    │       ├── components/ui/     # 46+ shadcn/ui components
    │       ├── components/        # AppShell, Sidebar, TopBar, QueryGuard, FormField, etc.
    │       ├── stores/            # authStore, uiStore (Zustand + persist + devtools)
    │       ├── lib/               # apiClient, useSSE, useZodForm, format utils
    │       └── index.ts           # Barrel export (all public API)
    └── config/                    # Shared tailwind.config.ts, tsconfig.json, .eslintrc.js
```

### Key Store API

```typescript
// authStore — persisted as 'eusolicit-client-auth-store' / 'eusolicit-admin-auth-store'
const { user, token, isAuthenticated, login, logout, setTokens } = useAuthStore();

// uiStore — sidebarCollapsed + locale persisted; toasts in-memory
const { sidebarCollapsed, locale, addToast, removeToast } = useUIStore();

// useToast() shorthand
const { toast } = useToast();
toast.success('Saved'); toast.error('Failed'); toast.info('Loading');
```

### Key Component API

```typescript
// Data fetching guard
<QueryGuard isLoading={isLoading} isError={isError} isEmpty={!data?.length}>
  <MyList data={data} />
</QueryGuard>

// Form pattern
const form = useZodForm(mySchema);
<FormField name="email" label={t('forms.email')} control={form.control}>
  <Input type="email" />
</FormField>
```

---

---

## Analytics & Admin Platform Reference

_Quick reference for agents working on post-E12 analytics, admin, or enterprise API concerns._

### Analytics Materialized Views

Five MV domains refreshed by Celery Beat in the `notification` service:

| View | Refresh | Source Tables |
|------|---------|--------------|
| `client.mv_market_analytics` | Daily | `bid_decisions`, `opportunities` |
| `client.mv_roi_analytics` | Daily | `bid_preparation_logs`, `bid_outcomes` |
| `client.mv_team_analytics` | Daily | `bid_preparation_logs` (per user) |
| `client.mv_competitor_analytics` | Daily | `competitor_records` |
| `client.mv_usage_analytics` | Hourly | `usage_meters` |

**Critical:** All MVs use `REFRESH MATERIALIZED VIEW CONCURRENTLY` — requires unique index on MV filter columns. Without the unique index, `CONCURRENTLY` fails silently and falls back to locking refresh. Never drop the `ix_mv_*` indexes.

### Tier Access Matrix (Analytics)

| Dashboard | Free | Starter | Professional | Enterprise |
|-----------|------|---------|--------------|------------|
| Market Intelligence | ❌ | ✅ | ✅ | ✅ |
| ROI Tracker | ❌ | ✅ | ✅ | ✅ |
| Team Performance | ❌ | ✅ | ✅ | ✅ |
| Usage Dashboard | ❌ | ✅ | ✅ | ✅ |
| Competitor Intelligence | ❌ | ❌ | ✅ | ✅ |
| Pipeline Forecasting | ❌ | ❌ | ✅ | ✅ |

Tier gate middleware returns `403 {"error": "tier_required", "minimum_tier": "professional", "upgrade_url": "/billing"}`.

### Admin API (VPN-Restricted)

- **Routing:** Internal service at `admin-api/`, VPN/IP allowlist via `IPAllowlistMiddleware`
- **Auth:** Admin JWT with `role: admin` claim — different from client API JWTs
- **Endpoints:** `/admin/tenants/*`, `/admin/crawlers/*`, `/admin/tenants/{id}/white-label`, `/admin/audit-logs/*`, `/admin/analytics/*`
- **IP Allowlist:** Configured via `ADMIN_IP_ALLOWLIST` env var (comma-separated CIDRs); test with `test_ip_allowlist.py`

### Enterprise API

- **Routing:** `api.eusolicit.com/v1/*` proxies to Client API endpoints
- **Auth:** `X-API-Key` header validated against SHA-256 hash in DB; associated with `company_id` and tier
- **Rate Limiting:** Redis token bucket per API key; rate limits vary by tier; 429 + `X-RateLimit-*` headers
- **API Key Storage:** Full key shown exactly once at creation; only SHA-256 hash stored in DB (`enterprise_api_keys` table)
- **Docs:** Swagger UI at `/v1/docs`, Redoc at `/v1/redoc` (auto-generated from FastAPI routes)

### Report Generation

- **Engine:** reportlab (PDF) + python-docx (DOCX) — same stack as E07 proposal export (shared code)
- **Async:** Generated via Celery task; output stored in S3 with 24h signed URL
- **Types:** pipeline_summary, bid_performance_summary, team_activity, custom_date_range
- **Scheduled:** Celery Beat triggers weekly (Monday) or monthly (1st) per company admin config; delivered via SendGrid attachment
- **On-demand:** `POST /reports/generate` → job ID → poll `GET /reports/jobs/{job_id}` → download URL when ready

---

## Epic 22 Patterns — On-Prem Launch (ADR-010 Pivot)

_Added by Epic 22 retrospective — 2026-05-12 (corrected: E22 retro claimed these writes but did not execute them; written now by E23 retro)._

- **Architectural pivots must follow the ADR-rewrite + sprint-change-proposal pattern — never in-flight story mutation.** (E22-P01, SEVERITY: high) When a major architectural decision reverses mid-sprint (e.g., AWS → single-host Docker per ADR-010), the correct sequence is: (1) rewrite the ADR with the new decision and rationale; (2) author a decision record document; (3) mark superseded stories `done` with `SUPERSEDED` banners pointing to replacement stories; (4) inject new epic via `bmad-correct-course`. Never mutate in-flight stories with the new technology approach — this leaves hybrid code that's neither the old nor the new approach. Reference: E22 ADR-010 pivot; pe-02/pe-03 marked `done` (SUPERSEDED); E22 + E23 epic injection via sprint-change-proposal-2026-05-12.

- **Single shared `_lib.sh` for sibling infra scripts — DRY principle applied to operator automation.** (E22-P02, SEVERITY: medium) When multiple infra scripts in the same domain (backup, restore, monitoring) share logic (credential loading, structured logging, error exits), extract shared code to `_lib.sh` sourced by all siblings. Ensures: consistent log format for `grep`-ability, single credential-path definition, uniform exit-code semantics. All 3 backup scripts in E22 onprem-01/02 follow this pattern (`scripts/onprem/_lib.sh`). Inconsistent error handling in sibling scripts is the primary cause of silent backup failures.

- **Redis `maxmemory-policy` must be `noeviction` when Redis holds non-cache-only state (Celery broker, Streams consumer groups).** (E22-P03, SEVERITY: high) `allkeys-lru` is correct for pure-cache Redis. When Redis also serves as Celery broker or event-bus (Redis Streams), `allkeys-lru` silently evicts task/event state under memory pressure without error — the eviction is logged at DEBUG level only. The correct policy is `noeviction` — Redis returns an error on new writes when memory is full, which surfaces the problem explicitly rather than silently dropping task/event state. Reference: E22 onprem-02 correctness fix (changed from `allkeys-lru` to `noeviction` before launch).

- **Runbook coverage CI lint gate extends naturally from app-layer to host-layer alerts without script modification.** (E22-P04, SEVERITY: medium) The `check_runbook_url_coverage.py` gate is topology-agnostic — it checks `runbook_url` annotations in any Prometheus rule file under `infra/observability/`. Adding host-level alerts to a new `host-alerts.yaml` file automatically falls under the same lint gate. No changes to the gate script needed; just author the runbooks and the compounding gate works. Reference: E21 PE.06 gate extended to cover 10 new host-level alerts in E22 onprem-04.

- **Deploy pipeline CI-gate must use `workflow_run` trigger, not parallel `on: push` — structural fix for auto-sync outage class.** (E22-P05, SEVERITY: high) Running CI and deploy workflows in parallel on `push: branches: [main]` means a failing commit deploys before CI catches it. Fix: change `deploy.yml` to `on: workflow_run: workflows: [CI]: types: [completed]` with `if: github.event.workflow_run.conclusion == 'success'`. Add auto-rollback on smoke-test failure. This makes the CI gate structural (not advisory) and eliminates the "auto-sync ships broken code to prod" failure mode. Reference: E22 onprem-05 — closed the 2026-05-09 `chore: auto-sync ac2fd08` 4-hour outage structurally.

- **Operator-execution stories distinguish CODE-AC (reviewable) from LIVE-AC (requires hardware execution).** (E22-P06, SEVERITY: high) Infrastructure stories where ACs include "execute a drill and populate §First Drill Results" cannot be code-reviewed to `done` without the drill having been executed. Define two AC categories per story: CODE-AC (all Terraform/script/config code merged + CI-passing; sprint-status transitions to `review`) and LIVE-AC (operator execution complete + evidence committed; sprint-status transitions to `done`). Without this distinction, a story can be `review`-approved but not actually `done`. Reference: All 6 E22 onprem stories have explicit PENDING OPERATOR ACTION sections separating CODE-AC from LIVE-AC.

- **On-prem infrastructure stories must resolve pre-existing resource pressure as a story entry criterion, not a post-deployment follow-up.** (E22-P07, SEVERITY: high) When the deployment target has a resource constraint (e.g., `/` at 85%, Docker image growth expected), resolving that constraint is a hard prerequisite for the story — not a nice-to-have follow-up. Story entry criteria for onprem-03: "Confirm `/` < 75% after Docker storage-root migration to `/home/docker/`." Deploying new Docker containers (Prometheus, Grafana, Alertmanager) onto a host at 85% disk will breach the 90% critical threshold mid-deployment and trigger an alert storm during setup. Reference: E22 onprem-04 finding; Docker storage-root migration (daemon.json to `/home/docker/`) added as E22-A3 pre-requisite.

### Epic 22 Anti-Patterns

- **Don't treat PM sprint-change-proposal "dev passes" as formal story completions — stories still require full bmad-dev-story → bmad-code-review cycle.** (E22-AP01, SEVERITY: critical) When the PM injects a sprint-change-proposal with "LANDED IN CODE" notes, the code has been sketched/outlined but has NOT gone through the formal dev-story gate. Sprint-status `ready-for-dev` is accurate — these stories require a full bmad-dev-story session (ATDD, implementation, test runs) and bmad-code-review before closure. Treating PM seed passes as story completion is a false-done anti-pattern. Reference: All 6 E22 onprem stories had PM seed passes; none had ATDD checklists or formal test results.

- **Don't conflate "code done" and "live done" for operator-execution epics — the gap between them is the widest in E22 of any prior epic.** (E22-AP02, SEVERITY: high) For epics where the acceptance criteria include "execute a drill and populate §First Drill Results," the story cannot close `done` without the drill having been executed. The code scaffolding is a prerequisite for the live execution, not the deliverable itself. The distinction matters for: (a) entry criteria (operator must have hardware access), (b) acceptance evidence (populated §Results sections, not CI test results), (c) definition of `done` (operator attestation, not just Approve verdict). Reference: E22 onprem-01..06 all require drill execution or hardware provisioning to close.

- **Don't skip Docker storage-root migration when deploying container-heavy stacks to a host at >75% disk capacity.** (E22-AP03, SEVERITY: high) Deploying Prometheus, Grafana, Alertmanager, and exporters onto a host where `/` is at 83–85% used will breach the 90% critical threshold and trigger an alert storm during the very deployment that enables alerting. Always resolve disk pressure first: migrate Docker storage root to a separate partition (`/home/docker/` via `daemon.json`), confirm `/` < 75%, then deploy the observability stack. This is a story entry criterion, not a post-deployment cleanup item.

- **Don't accumulate TEA review debt in a separate backlog story (`inj-03`) — embed TEA as a mandatory dev-story AC.** (E22-AP04, SEVERITY: critical — 17th consecutive epic) The `inj-03` TEA review backlog approach has failed for 17 consecutive epics. See AP21-03 (16th-epic entry). The 17th recurrence escalates this to a structural intervention requirement: the `bmad-dev-story` template must be modified to include TEA review as a required AC before any Phase-2 story is dispatched.

---

## Epic 23 Patterns — Operational Close-Out

_Added by Epic 23 retrospective — 2026-05-12._

- **Operator-execution stories must gate on upstream live-state evidence, not upstream code-gate completion.** (E23-P01, SEVERITY: high) When operator stories (drills, provisioning, soak gates) depend on infrastructure from a prior epic, the dependency must be expressed as "upstream LIVE (drills executed, services running on hardware)" not "upstream `done` (code merged)." Sprint-status `done` on an infrastructure story means CODE GATE. LIVE GATE is separate and may be hours/days later. Expressing the dependency as code-gate creates false-ready signals: E23 pe-04 cannot run chaos drills against alertmanager that isn't provisioned, even if onprem-03 is sprint-status `done`. Always check: "is the prerequisite infrastructure actually live on the target host, or just code-reviewed?"

- **SLA disclosure gates use calendar-based soak periods — publish a conservative posture first, upgrade only after observation.** (E23-P02, SEVERITY: medium) The correct launch sequence for SLA commitments: (1) Deploy "Beta — best-effort availability" posture card on Trust Center with internal RTO/RPO targets; (2) Observe 2-week soak period with zero incidents or incidents handled within stated RTO; (3) Only then publish a numeric SLA or uptime percentage claim. Publishing a numeric SLA at launch before production patterns are established creates an immediate breach risk. The `public-sla-announcement-soak-gate` story demonstrates this correctly — `public-sla-soak` is a calendar gate, not a code gate.

- **Rescope a story when technology changes but the operational goal survives — preserve story ID, history, and audit trail.** (E23-P03, SEVERITY: medium) When an architectural pivot changes the implementation technology (e.g., PagerDuty → email+Telegram per ADR-010), the operational goal (verifying on-call paging) survives. Rescope the story: update Title, §Implementation Decision, and ACs; preserve the Story ID, source reference, and acceptance history. Cancelling the story and creating a new one severs the audit trail and loses the rationale for the original design. Reference: pe-06 rescoped 2026-05-11 from PagerDuty provisioning to email+Telegram alertmanager verification.

### Epic 23 Anti-Patterns

- **Don't treat TEA backlog as a separate story — 18 consecutive epics without TEA score is a terminal quality risk.** (E23-AP01, SEVERITY: critical — 18th consecutive epic) The `inj-03` TEA review backlog story has been at `ready-for-dev` since Epic 13 (Epics 8–23 = 18 consecutive epics without TEA score). The separate-story approach cannot succeed — it is always last in the dispatch queue. The only mechanical fix: embed TEA review as a mandatory acceptance criterion in the `bmad-dev-story` phase prompt template with format: "TEA review executed; score ≥ 80/100 documented in Dev Agent Record; or blocking findings enumerated with remediation plan." Retire `inj-03`. This is the final retro warning; Phase-2 must not begin without this structural fix in place.

- **Don't inject operator-execution stories into an epic without adding individual sprint-status keys — granular dispatch and completion tracking is required.** (E23-AP02, SEVERITY: high) When a sprint-change-proposal creates an operator-execution epic, each story must receive its own `development_status` key in `sprint-status.yaml`. E23 has only `epic-23: in-progress` — no keys for the 5 individual stories. Operator stories (pe-04, pe-06, public-sla-soak) need individual `done` signals when operators complete each drill/provisioning/soak. Without story-level keys, the orchestrator has no dispatch handles and the operator has no way to signal per-story completion. All 5 E23 story keys must be added surgically.

- **Don't accept partial AC implementation ("zombie code") as satisfying an acceptance criterion — complete all enumerated targets in a single dev-story pass.** (E23-AP03, SEVERITY: high) A PM seed pass or partial dev session that wraps 1 of 9 Stripe call sites, or registers metrics without wiring emissions, creates zombie code: code that appears to satisfy an AC on superficial review but fails acceptance testing. The story AC must enumerate all targets explicitly (e.g., list all billing_service.py line numbers with Stripe SDK calls). A formal bmad-dev-story session must complete all targets before the story enters review. "Partially wrapped" is not "wrapped." "Metrics registered but not emitting" is not "metrics live." Zombie code passing code review is a quality defect.

- **Don't claim project-context.md updates in sprint-status without verifying the file was actually written — zero-output guard must cross-check file state.** (E23-AP04, SEVERITY: high) A bmad-retrospective session claimed "project-context.md updated with 8 E22 pattern/anti-pattern entries" in the sprint-status comment (epic-22-retrospective 2026-05-12), but static inspection confirmed zero E22 entries in the file. This is the zero-output guard failure mode: a session emits a completion claim for file writes that did not occur. The cross-check rule: before any completion claim involving file writes, enumerate which file paths were written and confirm the Write tool was called in that session. "Intended to write" and "actually wrote" are different outcomes.

---

## Epic 4 Amendment Patterns — SirmaAI Refactor (ai-gateway → sirmaai-gateway)

_Added by Epic 4 SirmaAI Amendment retrospective — 2026-05-14. Covers stories S04.20–S04.31._

- **Compose circuit-breaker keys from business semantics, not auto-derive them from the client library.** (E04A-P01, SEVERITY: medium) For external calls with a per-tenant identity dimension (SirmaAI Project ID, OAuth installation ID, MCP workspace ID), the circuit key MUST be `f"{operation}:{logical_name}:{tenant_id}"` composed by the caller (router endpoint) and passed as a keyword argument to the resilient client. Bare `logical_name` keys collapse tenant outages into a single circuit — one Project's failure trips siblings. Reference: `sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_async_client.py` `submit_async(..., circuit_key: str)`. Anti-reference: S04.28 Round-1 code used `circuit_key=exc.agent_name` (bare); caught in code review and fixed in Round 3.

- **Route every write to a shared row through one SQL-guarded `converge()` repository method.** (E04A-P02, SEVERITY: high) When multiple callers (foreground poll + inbound webhook + reconciler Beat task) write to the same row, idempotency is correct-by-design only if all three call one repository method with a `WHERE status IN (<non-terminal-values>)` guard. Application-layer locks/mutex are fragile under concurrent access. Reference: `WorkflowRunRepository.converge()` introduced in S04.24 and reused in S04.25 + S04.26. Pattern to apply in E28 (Webhook & Run-State Reconciliation) and E26 (Agent-Driven Ingestion analysis results).

- **Type every per-tenant secret as `SecretStr` end-to-end with a structured-log redaction assertion test.** (E04A-P03, SEVERITY: high) Every per-tenant secret (API key, HMAC secret, OAuth token, refresh token, MCP bearer) must be (a) typed `SecretStr` at the DB read boundary, (b) unwrapped via `.get_secret_value()` exactly once at the HTTP call site and never reassigned to a `str` binding, (c) covered by a test using `structlog.testing.capture_logs()` that asserts no captured log entry contains the plaintext. The `error=str(exc)` pattern in except-blocks is a known leak risk — log `error_type=type(exc).__name__` instead. Reference: `sirmaai-gateway/src/sirmaai_gateway/services/key_vault.py` + `tests/unit/test_key_vault.py`.

- **Inbound webhooks follow the Standard Webhooks spec verbatim, including multi-value signature tolerance during rotation overlap.** (E04A-P04, SEVERITY: medium) Receiver MUST: (1) read raw bytes via `await request.body()` BEFORE JSON parsing; (2) build canonical HMAC input as `f"{webhook_id}.{webhook_timestamp}.".encode() + raw_body` (length-prefixed bytes to prevent collision attacks); (3) compare via `hmac.compare_digest()` (project rule 48); (4) tolerate `webhook-signature: v1,<sig1> v1,<sig2>` (space-separated multi-value) during rotation overlap; (5) store idempotency key (`webhook-id`) in Redis SETNX with TTL ≥ provider's retry window (7 days for SirmaAI); (6) DLQ poison events to a dedicated table with full payload + `error_type`. Reference: `sirmaai-gateway/src/sirmaai_gateway/routers/webhooks.py`.

- **Long-running external operations follow the reconciler-as-authoritative-truth eventual-consistency pattern.** (E04A-P05, SEVERITY: high) Architecture: (a) Submit async + persist `workflow_runs` row in `pending`; (b) Inbound webhook converges via `repository.converge()` (latency optimisation; may be lost); (c) 5-min Celery Beat reconciler scans non-terminal rows, polls source-of-truth API, also calls `repository.converge()`; (d) Foreground GET reads the row. The reconciler — not the webhook — is authoritative truth (architecture amendment 2026-05-12 §4.4). RPO = 5 min. Apply to E26 (Agent-Driven Ingestion), E28 (Webhook & Run-State Reconciliation), and any future event-driven epic. Reference: `sirmaai_gateway/services/reconcile_runs_task.py`.

- **Public-ingress cleanups delete the dangerous surface in the same commit as the new narrow surface.** (E04A-P06, SEVERITY: medium) When changing a service's public exposure, delete the old broad surface (e.g., nginx `location /ai/ {...}`) in the SAME commit that adds the new narrow literal-path surface (e.g., `location = /webhooks/sirmaai {...}`). Co-locate runbook + cert SAN expansion. Co-existing broad + narrow surfaces during cutover is the highest-risk attack window of any deploy. Reference: S04.29 nginx vhost diff (`infra/nginx/api.eusolicit.com.conf`).

- **Cross-tenant negative tests embedded in AC templates by default — not added at code-review time.** (E04A-P07, SEVERITY: medium) Every amendment story from S04.23 onward shipped with an explicit cross-tenant negative test as an AC, not as a code-review afterthought. Examples: `test_resolver_does_not_leak_across_companies` (S04.23 AC 10), `test_get_status_does_not_leak_across_companies` (S04.24 AC 9), HMAC-validity-cross-subscription test (S04.25 AC 7). The resolver's composite key `(eusolicit_company_id, sirmaai_project_id)` makes cross-tenant leakage structurally impossible. **This is the single strongest tenant-isolation discipline demonstrated to date** and should be the template for all multi-tenant stories.

### Epic 4 Amendment Anti-Patterns

- **Don't accept "I'll fix it in the next round" for HIGH code-review findings — single-shot final-patch discipline is mandatory.** (E04A-AP01, SEVERITY: critical) When a story enters re-review after Round 1, ALL findings must be committed BEFORE re-submission. S04.26 went through 4 review rounds with the same `CircuitOpenError silently ACKs` HIGH finding repeatedly carried as "in the working tree but not committed." This inflates review surface, fragments reviewer attention across the same code areas, and delays closure. The dev-story prompt must require: "Before requesting re-review, list every prior-round finding and the commit SHA where it was fixed. Re-review is rejected if any prior finding lacks a SHA."

- **Don't let story rows drift `in-progress` while downstream-dependent stories close.** (E04A-AP02, SEVERITY: high) `4-20-service-rename-and-flag-scaffold: in-progress` while S04.21–S04.31 (all of which depend on the rename) are `done` is a logical contradiction. Either the rename completed inline during a later story (S04.30 package rename most likely) and the row was never flipped, or there is genuine outstanding work being silently bypassed by the feature flag default `SIRMAAI_GATEWAY_ENABLED=false`. Add a sprint-status-lint CI rule: if all stories `S{epic}.{n}` for `n > N` are done and `S{epic}.{N}` is in-progress, raise a warning. This is a known recurrence — 5th consecutive sprint-status-drift anti-pattern (cf. AP21-04, E03 retro carry-forward).

- **Don't run NFR assessment reactively after the epic closes — execute it in parallel with the final third of stories.** (E04A-AP03, SEVERITY: high) The Epic 4 NFR report was generated 2026-05-14, after 11 of 12 amendment stories were `done`. Five evidence gaps surfaced (k6 baseline, SCA scan, CPU/memory profile, ATDD checklists, DR drill) that should have been parallel-executed during the amendment, not discovered at the end. The 2026-04-23 original-scope retro already flagged "no NFR assessment" — running it 21 days later replays the same pattern. Make `bmad-testarch-nfr` a mandatory pre-prod-flag-flip gate, executed alongside (not after) the final amendment/feature story.

- **Don't ship feature epics without ATDD checklists — 2nd recurrence in same epic phase (S04.01–S04.10 then S04.20–S04.31).** (E04A-AP04, SEVERITY: high) `ls eusolicit-docs/test-artifacts/atdd-checklist-4-2*` returns nothing. The 2026-04-23 retro explicitly committed to "Enforce ATDD checklist creation as a before-dev gate for every E05+ story" — yet 12 amendment stories shipped with zero ATDD checklists. Tests exist (61 files) and pass; the discipline gap is documentation of RED→GREEN sequence, not the tests themselves. Either generate retroactive ATDD checklists OR document the gap explicitly in S04.20 closure.

- **Don't carry security/observability hardening across 6 consecutive epics — it is now a terminal escalation.** (E04A-AP05, SEVERITY: critical) Prometheus `/metrics` endpoint, Dockerfile USER directive, Dependabot config, `User.is_active` in login+refresh, bcrypt run_in_executor, Redis rate-limiter fail-open, JWT audience/issuer claim verification, pnpm audit in CI — first flagged E01 NFR, now 6 epics overdue (cf. AP21-03 16th-epic / E23-AP01 18th-epic TEA debt pattern). The NFR report flags Prometheus `/metrics` as a HIGH blocker for prod flag flip. Hardening cannot be a separate-backlog approach — must be embedded as mandatory AC in `bmad-dev-story` template, same fix pattern as E23-AP01 for TEA.

- **Don't co-exist legacy and refactored implementations behind a flag without a concrete deletion date.** (E04A-AP06, SEVERITY: medium) `eusolicit-app/CLAUDE.md` says the legacy `ai-gateway/` folder "remains callable for 1 sprint, then deleted" — no calendar date pinned. Without a date the deletion drifts; co-existence is intentional only for the cutover window. Always pin: "delete on YYYY-MM-DD = prod flag flip date + 14 days." Add to sprint-status as `delete-legacy-ai-gateway: backlog` with the calendar date.

- **Don't let multi-round code reviews repeat the same 3 root causes — embed pre-review self-checklist in dev-story prompt.** (E04A-AP07, SEVERITY: high) Cluster analysis of Round-1 findings across the amendment: (1) bearer-token/HMAC-secret leak risk via `str(exc)` or `.decode()` (4 stories: S04.22, S04.23, S04.24, S04.25); (2) missing cross-tenant negative test (S04.21, S04.22 Round 1); (3) idempotency SQL guard weakened or unused (S04.26 multiple rounds). The dev-story prompt template must prepend a 3-item pre-review self-checklist:
    1. "Cross-tenant negative test exists and asserts company-A request to company-B resource returns 403/404."
    2. "Every secret typed `SecretStr` end-to-end; no `error=str(exc)` in except-blocks for any secret-handling path; structured-log capture test asserts no plaintext."
    3. "All writes to shared rows route through repository method with `WHERE status IN (<non-terminal>)` guard."
  Reviews would then focus on subtler concurrency bugs (the actual added-value of human review), not regression-prevention sweeps.

---

**Last updated:** 2026-05-14 (Epic 4 SirmaAI Amendment Retrospective — 7 new patterns [E04A-P01..P07] and 7 new anti-patterns [E04A-AP01..AP07] added. Covers stories S04.20–S04.31. Note: S04.20 still `in-progress` — this is a partial retrospective; full closure of E04 amendment pending S04.20 audit + S04.26 review re-pass. Original Epic 4 retro [2026-04-23, S04.01–S04.10] remains separate.)
