---
project_name: 'EU Solicit'
date: '2026-04-09'
last_updated_by: 'retrospective-epic-3'
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

---

## Known Technical Debt

Key items from deferred-work.md (126+ items across E01–E03 — see full list there):

| Category | Key Items | Priority | Source Epic |
|----------|-----------|----------|------------|
| **Security** | No `User.is_active` check in login+refresh; no JWT audience/issuer verification; bcrypt password truncation; no `max_length` on password fields; concurrent token rotation race; Dockerfiles run as root; ports on 0.0.0.0; no Dependabot; no OAuth CSRF state validation; AuthGuard no hydration timeout; Zustand persist keys not namespaced per app | **HIGH** | E01, E02, E03 |
| **Concurrency** | Token rotation race (no SELECT FOR UPDATE); accept_invite TOCTOU; last-admin TOCTOU; rate limiter INCR+EXPIRE non-atomicity; non-atomic DLQ move; multi-tab wizard sync | **HIGH** | E01, E02, E03 |
| **Performance** | Blocking bcrypt on async event loop; no k6/Lighthouse performance baseline (3 epics); no code-split validation; CPV JSON bundle size unvalidated | MEDIUM | E01, E02, E03 |
| **Observability** | No Prometheus /metrics endpoint; no frontend error reporting (Sentry); no structured logging in auth_service; no `x-request-id` correlation header; no Grafana/Loki stack; no pnpm audit in CI | **HIGH** | E01, E02, E03 |
| **Data Integrity** | No UNIQUE on entity_permissions; no UNIQUE on espd_profiles(company_id, version); email case not normalized; `stub-company-id` fallback in wizard | MEDIUM | E02, E03 |
| **Reliability** | Redis outage blocks all logins; RSA key loss invalidates all JWTs; no circuit breaker in apiClient for repeated 500/503 | **HIGH** | E02, E03 |
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

**Last updated:** 2026-04-09 (Epic 3 Retrospective)
