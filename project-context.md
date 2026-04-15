---
project_name: 'EU Solicit'
date: '2026-04-14'
last_updated_by: 'retrospective-epic-4'
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

### From Epic 4 Retrospective

- **Don't use naive string equality (`==`) for HMAC signature comparison.** `header_sig == computed_sig` is vulnerable to timing attacks — an attacker can determine the correct signature one byte at a time by measuring response time differences. Always use `hmac.compare_digest()`.

- **Don't skip ATDD checklists for backend epics.** Epic 4 produced zero `atdd-checklist-4-*.md` files across 10 stories. The ATDD cycle (RED tests before implementation) is the discipline that proves test-driven development — skipping it makes the test design a post-hoc audit rather than a development gate. ATDD checklist scaffold must be created before story development starts, not after.

- **Don't let circuit breaker state live only in memory in multi-replica environments.** In-memory `asyncio.Lock`-guarded state is per-process. When a service scales to N replicas, each instance has independent circuit state: one instance may reject while another continues to hammer a failing agent. Document the limitation at implementation time; do not defer the Redis migration decision beyond the first auto-scaling event.

- **Don't implement SSE proxy without a wall-clock latency benchmark.** Functional tests (events forwarded in order) do not verify the < 100ms per-event latency requirement. SSE latency is a user-experience guarantee — verify it with a controlled mock SSE source with known send timestamps and `time.monotonic()` comparison before marking the story done.

- **Don't close an epic without an NFR assessment when the epic introduces stateful resilience, outbound HTTP clients, or streaming.** Epic 4 had no NFR report. The circuit breaker per-instance gap (E04-R-005), SSE latency unverified (E04-R-002 residual), and Redis publish reliability (E04-R-003 monitoring backlog) are NFR concerns that compound over time if not formally assessed.

- **Don't defer the same security hardening items across more than two consecutive epics.** Items deferred from E01 that remained open through E04 (Prometheus /metrics, Dockerfile USER, Dependabot, bcrypt executor, User.is_active, Redis fail-open, JWT audience/issuer) are now 4 epics overdue. Each additional deferral increases integration risk. Cap deferral at 2 consecutive epics — after that, inject as a mandatory story in the next sprint.

### From Epic 12 Retrospective

- **Don't mark a story `done` if it has frontend ACs and no E2E spec file.** Absence of a spec file means the AC is unverified at the browser level. A backend API test for a frontend AC is not sufficient — it proves the API works, not that the UI renders correctly.
- **Don't treat a non-functional story (load test, security audit) as done when its output files contain only templates.** Placeholder dashes in load-test-results.md and unchecked boxes in security-audit-checklist.md are not evidence. These stories have hard output artifact requirements.
- **Don't generate Celery tasks without integration test coverage of their execution.** Testing the endpoint that enqueues a task is not the same as testing the task. The side effects (S3 upload, DB write, SendGrid call) must be verified end-to-end.
- **Don't leave `test.skip()` decorators in E2E specs at epic close without a named activation story.** RED-phase specs are a deliverable; their activation plan is a separate required deliverable.
- **Don't skip TEA reviews because a story is "fully covered" or "simple".** Test quality and isolation can only be validated by a formal TEA pass. High test counts do not guarantee test quality.
- **Don't leave `epic-N: in-progress` in sprint-status.yaml when all stories are `done`.** This is the 4th occurrence of this anti-pattern. It will be enforced by CI going forward.
- **Don't implement analytics queries without `WHERE company_id = :cid` scoping on every materialized view query.** Cross-tenant data leakage in analytics is an R12.1 (Score 6) risk. Every analytics service method must include a cross-tenant negative test in its test suite.
- **Don't use `REFRESH MATERIALIZED VIEW` without `CONCURRENTLY` on production tables.** Non-concurrent refresh takes an exclusive lock that blocks all reads during refresh. Always add a `UNIQUE` index on materialized view columns used for filtering before enabling `CONCURRENTLY`.

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

**Last updated:** 2026-04-14 (Epic 4 Retrospective — patterns, anti-patterns, and AI Gateway technical debt added)
