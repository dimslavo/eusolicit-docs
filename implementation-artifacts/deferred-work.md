# Deferred Work — EU Solicit

Tracked items deferred from code reviews and implementation. Each entry includes source, description, and rationale for deferral.

## Deferred from: code review of story 3-8-authentication-pages (2026-04-08)

- No AbortController in callback useEffect — The mount-only `useEffect` in `callback/page.tsx` calling `exchangeOAuthCode()` has no cleanup function. If the component unmounts during the API call (back button, navigation), `.then()` fires on stale state causing potential memory leak and double navigation. Address when stubs are replaced with real API calls at E02 integration. Source: S03.08 edge-case review.
- OAuth state parameter not validated for CSRF — `callback/page.tsx` defaults missing `state` search param to empty string instead of treating it as a CSRF failure. Stub implementation masks this; real OAuth CSRF protection (validate state against session-stored nonce) must be implemented at E02 integration. Source: S03.08 edge-case review.

## Deferred from: code re-review of story 3-8-authentication-pages (2026-04-08)

- OAuth provider error params not surfaced in callback — When an OAuth provider returns `?error=access_denied` instead of a `code`, the callback page shows the generic "Missing authorization code" message instead of the provider's error reason. The `error` and `error_description` query params are silently discarded. Implement provider error parsing when wiring up real OAuth at E02 integration. Source: S03.08 edge-case re-review.
- Callback page has no authenticated-user redirect guard — Unlike login, register, and forgot-password pages, the callback page does not check `isAuthenticated` before exchanging the code. An already-authenticated user arriving at the callback URL will have their session silently overwritten. The OAuth state CSRF validation (E02 integration) will address this scenario. Source: S03.08 edge-case re-review.
- EIK field does not trim whitespace before regex validation — A copy-pasted EIK with leading/trailing spaces (e.g., from a PDF or spreadsheet) fails the `^\d{9}(\d{4})?$` regex despite containing a valid numeric value. Add `.trim()` to the Zod schema's eik field in a future schema refinement pass. Source: S03.08 edge-case re-review.

## Deferred from: code review of story 1-2-docker-compose-local-development-environment (2026-04-06)

- `reset-db` Makefile target destroys ALL named volumes (pgdata, redisdata, miniodata, clamdata) via `docker compose down -v` but only restarts postgres and redis — MinIO data and ClamAV virus signatures are lost and not restored. Matches spec verbatim; future improvement to add `make reset-db-full` or selective volume removal.
- Dockerfiles run as root — no `USER` directive in runtime stage. Acknowledged in Story 1.1 review as production hardening; not in scope for local dev.
- `:latest` tags on `minio/minio`, `minio/mc`, `clamav/clamav` images — non-reproducible builds across team members and over time. Matches spec-specified images; pin versions when stabilizing for CI/staging.
- `minio-init` one-shot container has no `restart: on-failure` policy — if bucket creation fails transiently, it is not retried. Low probability for local dev; add retry logic or restart policy if observed in practice.
- Per-service PostgreSQL roles (`client_api_role`, `admin_api_role`, etc.) referenced in DATABASE_URL defaults do not exist yet — connections will fail until Story 1.3 adds the init SQL script. Forward-compatible mount point (`infra/postgres/init/`) is already in place.
- `env_file: .env` in docker-compose.yml will fail on fresh clone because only `.env.example` exists — standard Docker Compose workflow requires developer to copy `.env.example` to `.env`. Consider adding a `make setup` target or pre-flight check.
- Duplicate `COPY packages/` in Dockerfile runtime stage copies unused source alongside pip-installed packages — dead-weight image bloat. Pre-existing from Story 1.1.
- Two-step `pip install` in Dockerfile builder (packages first, then service) can produce non-deterministic dependency resolution — package versions may drift between steps. Pre-existing from Story 1.1; fix with constraints file or single install command.
- No `restart:` policy on application services — crashed services stay down until manually restarted. Acceptable for local dev; add `restart: unless-stopped` if dev stability becomes an issue.
- All 5 application services share Redis database index 0 (`redis://redis:6379/0`) — potential key collision risk when services mature and use overlapping key patterns. No services use Redis yet; revisit when implementing cache/session logic.
- All Docker Compose port mappings bind to `0.0.0.0` (all network interfaces) — Postgres, Redis, MinIO, and ClamAV are accessible from the network on non-isolated dev machines. Use `127.0.0.1:<port>:<port>` for production-like hardening.

## Deferred from: code review of story 1-3-postgresql-schema-design-role-based-access (2026-04-06)

- No `REVOKE` on `public` schema — PG16 already revokes CREATE on public by default, but USAGE remains. Defense-in-depth: add `REVOKE ALL ON SCHEMA public FROM PUBLIC` when hardening for staging/production.
- Role creation not idempotent for password rotation — `IF NOT EXISTS` skips role creation entirely if role exists, leaving stale passwords. Add `ALTER ROLE ... PASSWORD ...` after each block for production environments.
- No `updated_at` trigger on `shared.tenants` — `DEFAULT now()` only fires on INSERT. Add a `before_update` trigger when Alembic manages this table in Story 1.4+.
- Orphaned test tables on assertion failure — if test assertion fails between setup and cleanup, `_atdd_*` tables persist. Mitigated by `IF NOT EXISTS` on creation. Refactor to pytest fixtures with `yield` + unconditional teardown.
- No cross-schema SELECT denial test — AC6 tests only verify INSERT/UPDATE/DELETE denial. PG16 handles SELECT isolation by default (no USAGE granted). Add explicit SELECT denial tests when expanding security test suite.
- No `search_path` configured for service roles — defaults to `"$user", public`. Application code must use fully-qualified table names. Consider `ALTER ROLE ... SET search_path` when services go to production.
- `_DB_NAME` env var vs `TEST_DATABASE_URL` env var split-brain risk — integration tests use `TEST_DB_NAME` while conftest.py uses `TEST_DATABASE_URL`. Both default to `eusolicit_test`. Unify config approach when test infra matures.

## Deferred from: code review of story 1-4-alembic-migration-scaffold (2026-04-06)

- `get_url()` in all 5 env.py files returns empty string with no validation when both `DATABASE_URL` and `sqlalchemy.url` are unset — produces cryptic SQLAlchemy error. Add `RuntimeError` guard with descriptive message.
- No `connect_timeout` on migration engine `create_engine()` — migrations hang indefinitely if database is unreachable. Add `connect_args={"connect_timeout": 10}` for CI/operability.
- No advisory lock for concurrent migration safety — two simultaneous `upgrade head` runs could corrupt `alembic_version` table. Add `pg_advisory_lock` when concurrent CI pipelines are introduced.
- `psycopg2-binary` in main `dependencies` (not dev/optional) — psycopg2 maintainers recommend source-built `psycopg2` for production. Switch when containerizing for staging/production.
- `script.py.mako` template doesn't include `SCHEMA` constant or schema-targeting reminder — future migration authors may accidentally omit `schema=` parameter, creating tables in wrong schema. Add template comment or constant.
- Makefile `MIGRATION_DB_URL` not quoted in shell commands — passwords containing `&`, `;`, or spaces cause shell interpretation errors. Wrap in single quotes when hardening for production use.

## Deferred from: code review of story 1-5-redis-streams-event-bus-setup (2026-04-06)

- Non-atomic DLQ move (xadd then xack) in `EventConsumer.process_pending()` — crash between the two Redis operations produces duplicate DLQ entries on next sweep. At-least-once semantics by design per spec; use a Lua script for atomicity when DLQ reliability is critical.
- `process_pending` hardcoded `count=100` in `xpending_range` — messages beyond 100 in the PEL are not inspected per sweep. Add pagination (cursor-based loop) when production traffic could exceed this threshold.
- No per-entry error handling in `process_pending` loop — one Redis failure mid-loop aborts all remaining entries. Wrap per-entry logic in try/except with logging when hardening for production.
- `STREAMS` dict not referenced by `CONSUMER_GROUPS` — CONSUMER_GROUPS hardcodes stream key strings directly instead of referencing `STREAMS.values()`. Values match today but could drift. Refactor to use STREAMS references.
- No payload size validation before `xadd` — oversized payloads cause untyped Redis error. Add a configurable max payload size check.
- `block_ms` accepts negative values in `EventConsumer.consume()` — undefined Redis behavior for `BLOCK` with negative int. Add input validation guard.
- `xclaim` returns None data if message was MAXLEN-trimmed from the stream — `dict(None)` would crash `process_pending`. Add a None guard after xclaim when MAXLEN trimming is configured.

## Deferred from: code review of story 1-8-github-actions-ci-pipeline (2026-04-06)

- `ignore_missing_imports = true` set globally in mypy [pyproject.toml] — pre-existing pattern from Makefile `--ignore-missing-imports` flag. Masks misspelled imports and missing internal deps. Narrow to specific third-party modules when type-checking matures.
- Shared package `-k` test filter is fragile and collision-prone [ci.yml] — `-k eusolicit_common` matches any test whose nodeid contains the substring. Works for current `test_eusolicit_<pkg>_*.py` naming convention but risks cross-matching. Migrate to directory-based isolation or pytest markers when test volume grows.
- Pip cache key omits root pyproject.toml and transitive dep hashes [ci.yml] — cache key only hashes per-project pyproject.toml. Changes to root config or transitive deps won't invalidate cache. Performance optimization only; pip still installs correct versions on cache miss.
- mypy exclude regex `.*/tests/` misses top-level `tests/` directory [pyproject.toml] — the regex requires at least one character before `tests/`. Only affects broader mypy invocations in test.yml and quality-gates.yml, not per-project CI. Fix regex to `(^|.*/)tests/` when consolidating mypy config.
- `eusolicit-test-utils` not listed in service dev dependencies [services/*/pyproject.toml] — service conftest.py files import fixtures from `eusolicit_test_utils` but the package isn't in dev extras. Currently masked because no actual service test files exist (only `__init__.py`). Add to dev deps when writing first service tests.

## Deferred from: code review of story 2-1-database-schema-auth-identity-tables (2026-04-07)

- Type annotations use `Mapped[sa.UUID]`/`Mapped[sa.DateTime]` instead of `Mapped[uuid.UUID]`/`Mapped[datetime.datetime]` — Cosmetic; mypy may flag but functionally works. Defer to code quality pass.
- `onupdate=sa.func.now()` is ORM-only, not a DB trigger — Raw SQL updates bypass it. Already tracked from Story 1.3 (no `updated_at` trigger). Add DB-level trigger when hardening for production.
- No CHECK constraint ensuring at least one auth method (`hashed_password IS NOT NULL OR google_sub IS NOT NULL`) on `client.users` — Application-layer concern. Revisit in Story 2.2 (registration) or Story 2.6 (Google OAuth).
- No `relationship()` declarations on ORM models — ORM convenience only. Add when writing CRUD endpoints in Stories 2.2+.
- No unique constraint on `entity_permissions(user_id, company_id, entity_type, entity_id, permission)` — Duplicate grants possible without it. Revisit in Story 2.10 (RBAC middleware).
- No unique constraint on `espd_profiles(company_id, version)` — Versioning invariant not enforced at DB level. Concurrent inserts can produce duplicate versions. Revisit in Story 2.12 (ESPD CRUD).
- Missing FK indexes on `subscriptions.company_id` and `espd_profiles.company_id` — PostgreSQL does not auto-create indexes on FK columns. Performance optimization; defer to query tuning.
- Subscription table missing `created_at`/`updated_at` columns — Skeleton table per spec. Billing expansion in Epic 8 will add them.
- No unique constraint on `subscriptions.company_id` — Multiple subscriptions per company currently allowed. Billing design detail for Epic 8.
- `audit_log.user_id` is a soft reference (no FK) — Cross-schema FK intentionally avoided per spec. Orphaned user_ids accumulate after user deletion. Documented design tradeoff.

## Deferred from: code review of story 2-2-email-password-registration (2026-04-07)

- bcrypt silently truncates passwords at 72 bytes — No max password length enforced. A 200-char password is accepted but only the first 72 bytes are verified on login. Add `max_length=128` on password field and/or pre-hash with SHA-256 before bcrypt. Security hardening.
- Email sent before DB commit — `send_verification_email` fires inside `register()` before the dependency wrapper commits the transaction. With the stub (log-only), harmless. With a real email service, a failed commit means an email was sent for a non-existent user. Address when implementing real email service.
- Blocking `bcrypt.hashpw` on async event loop — bcrypt cost=12 takes ~200-400ms of CPU, blocking the event loop and all concurrent requests. Wrap in `asyncio.loop.run_in_executor()` for non-blocking operation. Performance optimization.
- Email case sensitivity not normalized — `User@Example.com` and `user@example.com` create separate accounts. Pydantic `EmailStr` normalizes the domain but preserves the local part's casing. Add `LOWER(email)` functional unique index or normalize on input. Design decision needed.
- `get_engine()` module-level singleton not thread-safe — Concurrent first access can create duplicate engines and leak connection pools. Add a threading lock or `asyncio.Lock`. Low probability in single-worker deployment.
- `database_url=None` causes cryptic error — If `CLIENT_API_DATABASE_URL` is not set, `create_async_engine(None)` raises an opaque `ArgumentError`. Add an explicit check with a clear error message. Pre-existing config weakness.
- ORM `Mapped[]` type annotations use SA column types instead of Python types — `Mapped[sa.DateTime]` should be `Mapped[datetime]`, `Mapped[sa.UUID]` should be `Mapped[uuid.UUID]`. Pre-existing pattern across all models (user.py, company.py, etc.). Codebase-wide fix.
- No structlog logging in `auth_service.py` — Service layer has no logging of registration events (success, failure, duration). Observability improvement.

## Deferred from: code review of story 2-3-email-password-login-jwt-issuance (2026-04-07)

- Missing `User.is_active` check in login query [`auth_service.py:178-186`] — The login JOIN query does not filter on `User.is_active`, so deactivated users can still authenticate and receive valid JWTs. Not specified in Story 2.3 acceptance criteria. Track as future security hardening story.
- Redis outage blocks all logins (no graceful degradation) [`auth_service.py:174-175`] — `rate_limiter.check()` and `record_failure()` propagate Redis `ConnectionError` as unhandled 500, causing complete login unavailability. Consider fail-open pattern for rate limiting with structured logging.
- Nondeterministic membership selection for multi-company users [`auth_service.py:184`] — `LIMIT 1` without `ORDER BY` returns an arbitrary membership for users in multiple companies. Acknowledged in dev notes as acceptable for MVP. Future multi-company login should add `company_id` to `LoginRequest` or deterministic ordering.
- Rate limiter INCR+EXPIRE non-atomicity [`core/rate_limit.py:52-55`] — Acknowledged in dev notes. Under extreme concurrency near key expiry, the counter could theoretically become immortal (no TTL). Fix with Redis pipeline or Lua script.
- ORM `Mapped[sa.UUID]`/`Mapped[sa.DateTime]` type annotations [`models/refresh_token.py`] — Pre-existing pattern across all models. Should be `Mapped[uuid.UUID]` and `Mapped[datetime]` per SQLAlchemy 2.0 best practices. Already tracked from Stories 2.1 and 2.2.
- Unbounded password length on `LoginRequest` and `RegisterRequest` [`schemas/auth.py`] — No `max_length` on `password: str` field in either schema. Enables bcrypt HashDoS (large payloads fed to CPU-intensive hash). Pre-existing from Story 2.2. Fix both schemas together with `max_length=128` or similar.
- 429 rate-limit response envelope inconsistency [`api/v1/auth.py:52-57`] — HTTPException wraps detail as `{"detail": {"error": ...}}` while all other errors use `{"error": ..., "message": ..., "details": null, "correlation_id": "..."}`. AC7 doesn't specify body format; Retry-After header is correct. Fix by using `JSONResponse` directly instead of `HTTPException`.
- Anti-enumeration test relies on full JSON body equality [`tests/api/test_login.py:644`] — `assert wrong_pw_resp.json() == nonexistent_resp.json()` works now because `correlation_id` is empty in test env. Will break if correlation-ID middleware is added. Fix by excluding per-request fields from comparison.

## Deferred from: code review of story 2-4-jwt-authentication-middleware (2026-04-07)

- No `audience`/`issuer` claim verification on JWT decode [`security.py:165`] — `jwt.decode()` validates only signature and expiration. No `audience` or `issuer` check. A JWT from another service sharing the same RSA key pair would be silently accepted. Story 2.4 spec does not mention audience/issuer. Track for security hardening sprint.
- Token revocation not implemented [`security.py:159`] — Revoked/logged-out tokens remain valid for their full 15-minute lifetime. Explicitly scoped to Story 2.5 with a TODO comment.
- Empty-string role accepted through middleware chain [`security.py:176-180`] — A JWT with `role=""` constructs a valid `CurrentUser`. Gets rank 0 and is denied by all `require_role` checks (fails closed), but `GET /me` echoes back the empty role. Will be addressed when claim validation is hardened (patch finding 1).
- No structlog logger in `security.py` for auth events [`security.py`] — Security-sensitive operations (token validation failures, role rejections) produce no structured log entries. Observability improvement, not a functional bug.

## Deferred from: code review of story 2-5-token-refresh-revocation (2026-04-07)

- Race condition in concurrent token rotation [`auth_service.py:refresh_token()`] — No `SELECT ... FOR UPDATE` or DB-level constraint prevents two concurrent requests from both successfully rotating the same valid token, forking the family and defeating breach detection. Requires pessimistic locking or a unique partial constraint on `(family_id, is_revoked=false)`.
- Missing `User.is_active` check during token refresh [`auth_service.py:308-314`] — `refresh_token()` step 6 queries `User + CompanyMembership` but never checks `User.is_active`. Deactivated users can refresh tokens for up to 7 days. Same gap exists in `login()` (tracked from Story 2.3).
- Non-deterministic membership selection in refresh [`auth_service.py:308-314`] — `LIMIT 1` without `ORDER BY` on the user + membership query means users with multiple company memberships get an arbitrary company_id/role. Same pattern used in `login()` (tracked from Story 2.3).
- Missing index on `client.refresh_tokens.family_id` [`refresh_token.py`] — `UPDATE ... WHERE family_id = ?` in breach detection performs sequential scan. No index defined in model `__table_args__` or migration. Performance concern as token volume grows.
- Logout revocation indistinguishable from rotation revocation [`auth_service.py:logout()+refresh_token()`] — Both `logout()` and `refresh_token()` set `is_revoked=True` without recording the reason. A replayed logged-out token triggers false-positive breach detection, producing noisy `auth.token_breach` audit entries. Distinguishing requires a `revocation_reason` column or separate `revoked_by` enum.

## Deferred from: code review of story 2-8-company-profile-crud (2026-04-07)

- Duplicated CPV validator across PutRequest and PatchRequest [`schemas/company.py:62-72,90-100`] — Identical `validate_cpv_sectors` method copy-pasted in both schemas. Pydantic v2 class-method validators cannot easily be shared without a mixin. Cosmetic DRY concern; both are in the same file so divergence risk is low.
- `setattr` in PATCH loop has no allowlist of mutable fields [`company_service.py:119`] — The `else` branch of the PATCH field loop calls `setattr(company, field_name, ...)` for any field in `model_fields_set` not named `address` or `certifications`. Currently safe because schema fields match ORM columns, but a future schema-ORM mismatch would silently create transient Python attributes that don't persist. Consider adding a `_PATCHABLE_FIELDS` frozenset.
- No concurrency control — lost update race condition [`company_service.py:66,108`] — No optimistic locking (`version_id_col`) or `SELECT FOR UPDATE` on company row reads. Concurrent PUT/PATCH requests can overwrite each other's changes (classic lost update). Architectural concern affecting all CRUD operations in the codebase.
- Empty PATCH body creates no-op audit log entry [`company_service.py:100-138`] — AC6 says "that modifies"; an empty `model_fields_set` produces identical `before`/`after` audit entries. Story 2.11 audit middleware will likely address this.
- Missing size limits on `cpv_sectors`/`regions`/`certifications` lists and `description` length [`schemas/company.py`] — No `max_length` on JSONB list fields or `description`. Unbounded payloads risk storage bloat. Defense-in-depth concern not specified in story ACs.
- `ip_address` from `request.client.host` may be proxy IP [`api/v1/companies.py:43`] — Pre-existing pattern shared with `auth.py`. Behind a reverse proxy, audit logs record the proxy IP. Proxy header handling is a codebase-wide concern.
- Empty list `[]` vs `None` semantic ambiguity in JSONB columns [`schemas/company.py`] — `cpv_sectors: []` stored as JSON array vs `None` as SQL NULL. Downstream code checking `IS NULL` vs `= '[]'::jsonb` will behave differently. Not specified in story ACs.

## Deferred from: code review of story 2-9-team-member-management (2026-04-07)

- `bcrypt.hashpw` blocks async event loop in `accept_invite` [`member_service.py:406-408`] — CPU-bound bcrypt at rounds=12 (~200ms) runs in async context without executor offload. Pre-existing pattern from auth_service registration flow (Story 2.2).
- Concurrent `accept_invite` race condition (TOCTOU) [`member_service.py:355-431`] — Two simultaneous requests with the same token can both pass validation before either sets `accepted_at`. Second request hits IntegrityError → 500. Fix with `SELECT ... FOR UPDATE` on invitation row. Pre-existing pattern in token flows.
- `remove_member` TOCTOU on admin count [`member_service.py:283-303`] — Admin count read before membership fetch. Concurrent admin removals could both pass the last-admin check. Extremely low probability in practice.
- Invitation model `Mapped[sa.DateTime]` type hints [`invitation.py:40-47`] — Should be `Mapped[datetime]` for proper static analysis. Pre-existing project-wide convention tracked since Story 2.1.
- No password complexity validation in `accept_invite` [`member_service.py:400-404`] — Password only checked for truthiness; whitespace-only strings pass. Cross-cutting concern that should be unified with registration/password-reset flows.

## Deferred from: code review of story 2-10-entity-level-rbac-middleware (2026-04-07)

- Bypass path (admin/bid_manager) executes 2 sequential queries instead of a single optimised query [`rbac.py:247-283`] — `own_company_stmt` and `other_company_stmt` run as separate `session.execute()` calls. Introduces TOCTOU race between queries. Could be consolidated into `SELECT company_id FROM entity_permissions WHERE entity_type=:et AND entity_id=:eid LIMIT 1` and branched in Python. AC5 technically requires a single query; bypass path is admin-only and low-frequency.
- TestSingleQueryConstraint only covers the non-bypass path [`test_rbac_middleware.py:656-704`] — The AC5 test exercises contributor role only. The bypass path's 2-query behavior is untested. Related to the bypass query consolidation above.
- `test_denial_audit_write_does_not_block_403_response` does not simulate flush failure [`test_rbac_middleware.py:811-836`] — The test only verifies normal 403 flow. Does not mock `_write_denial_audit` to raise and verify 403 is still returned. Depends on the audit write try/except fix being applied first.
- Bypass roles access unclaimed entities (zero entity_permissions rows) [`rbac.py:280-283`] — Design decision: when no entity_permissions rows exist for any company, bypass roles (admin/bid_manager) are granted access. AC4 says bypass applies "for entities that belong to the current user's own company." An entity with zero rows doesn't demonstrably belong to anyone. Acceptable if entity creation always seeds initial permissions; revisit if entities can exist without rows.
- Mixed-company entity_permissions rows bypass cross-company isolation for bypass roles [`rbac.py:247-256`] — If an entity has entity_permissions rows for BOTH Company A and Company B (data anomaly), the own-company check passes immediately without detecting the multi-company state. Depends on data integrity; the schema has no UNIQUE constraint preventing this. See deferred-work entry for Story 2.1 (no unique constraint on entity_permissions).
- No UNIQUE constraint on (user_id, company_id, entity_type, entity_id) in entity_permissions — Duplicate rows would cause `scalar_one_or_none()` to raise `MultipleResultsFound` (500 error). Schema change outside Story 2.10 scope. Future grant API story should enforce uniqueness at application layer and consider adding a DB-level UNIQUE constraint. Pre-existing from Story 2.1.
- No integration test for `entity_id_param` override — Factory accepts the kwarg and unit test verifies acceptance, but no integration test exercises a route with a non-default path parameter name. Can be addressed when a route in E03/E06 uses a custom param name.
- Bypass path queries lack covering index [`rbac.py:247-264`] — Bypass queries filter on `(entity_type, entity_id, company_id)` which doesn't match the composite index `(user_id, entity_type, entity_id)`. Consider adding `ix_entity_permissions_entity_lookup(entity_type, entity_id, company_id)` when table grows.

## Deferred from: code review of story 2-12-espd-profile-crud (2026-04-07)

- No size/depth limit on `field_values` JSONB payload — clients can submit arbitrarily large JSON into the freeform `dict` sections. Pre-existing architectural decision (Dev Note 17: "do NOT try to validate nested keys"). Already tracked as "no payload size validation" (LOW priority). Add configurable max payload size when hardening for production.
- No-op PATCH (empty `field_values: {}` or all-None fields) creates phantom version rows with identical content to the previous version, inflating version history and producing misleading "update" audit entries. Not a spec violation (AC3 does not prohibit no-op patches). Add `if not patch_dict: raise HTTPException(422, "No fields to update")` guard when UX feedback warrants it.
- `IntegrityError` on concurrent version collision (UNIQUE constraint on `company_id, version`) propagates raw SQLAlchemy error details in 500 response, leaking table/constraint/column names. Intentional per Dev Note 7 ("let the DB error propagate"). Add `try/except IntegrityError → HTTPException(409, "Version conflict, please retry")` when concurrent write patterns become common.
- Redundant queries in `upsert_espd_profile_full` and `upsert_espd_profile_partial` — `_get_latest_or_none` (SELECT ORDER BY version DESC LIMIT 1) and `_get_next_version` (SELECT MAX(version)) query the same table for the same company_id. Could derive version from `existing.version + 1` or `1` if None, eliminating one round-trip. Minor optimization.
- Pre-existing from Story 2.1: `models/espd_profile.py` missing `from __future__ import annotations` header. Also `onupdate=sa.func.now()` on `updated_at` column is dead code in append-only design (rows are never updated). Model was not modified in this story.
- No test assertion for AC7 `entity_id` — ESPD audit tests in `test_audit_trail.py` verify `entity_type`, `action_type`, `before`/`after`, `user_id`, and `ip_address`, but do not assert `audit_log.entity_id` equals the newly created `espd_profiles` row UUID. Implementation correctly passes `entity_id=new_profile.id`. Add assertion when expanding audit test coverage.

## Deferred from: code review of story 3-1-next-js-14-monorepo-scaffold (2026-04-08)

- `turbo: "latest"` in root `frontend/package.json` devDependencies — every fresh `pnpm install` can resolve a different major version of Turborepo, risking non-reproducible builds and silent CI breakage. Spec-compliant; pin to a specific version (e.g., `^2.9.0`) when stabilizing for CI.
- `@eusolicit/config` package has no `exports` field — subpath imports (`tailwind.config`, `prettier.config.js`, `tsconfig.json`) rely on pnpm's internal symlink filesystem traversal. Will break under strict ESM `exports` enforcement or alternative package managers. Add an `exports` map when migrating to ESM.
- `@/*` TypeScript path alias maps to `./src/*` but no `src/` directory exists in either app — the alias is unused scaffold. Developers attempting `@/` imports will get module resolution errors. Align alias target with actual directory structure (e.g., `./*` or `./app/*`) when app code grows beyond placeholders.
- UI package (`@eusolicit/ui`) uses raw TypeScript entrypoint (`"main": "./index.ts"`) with no build step — works via `transpilePackages` in Next.js but will fail for non-Next consumers (Jest, Storybook, Vite). Add a build step and compiled output when non-Next consumers are introduced.
- `@typescript-eslint/eslint-plugin` and `@typescript-eslint/parser` pinned to `^6.0.0` (end-of-life) — no longer receives patches and blocks migration to ESLint 9 flat config. Upgrade to v7+/v8+ when addressing linter modernization.
- `engines.pnpm >=8` inconsistent with `packageManager: pnpm@10.33.0` — engines field implies pnpm 8/9 are acceptable but the lockfile format (v9) requires pnpm 9+. Tighten engines to `>=9` or `>=10` to match actual requirements.
- Tailwind config shallow spread (`...baseConfig`) in app configs silently discards base config additions — local `content` key overwrites (not merges) any future base `content` entries. Story 3.2 (Tailwind design token preset) should establish a proper deep-merge pattern.
- No `.gitignore` in `frontend/` directory — `.next/`, `node_modules/`, `.turbo/`, `tsconfig.tsbuildinfo` should be gitignored. May rely on parent-level gitignore; verify coverage before first commit.

## Deferred from: code review of story 3-4-responsive-layout-strategy (2026-04-08)

- BottomNav uses `<div>` root instead of `<nav>` semantic element — screen readers cannot identify it as a navigation landmark; Sidebar and MobileSidebarSheet both use `<nav>` correctly. Trivial fix: change outer `<div>` to `<nav>` with `aria-label="Bottom navigation"`. Spec did not require it; defer to accessibility audit.
- Safe-area padding on notched devices may compress BottomNav content — `h-16` with `box-sizing: border-box` + `paddingBottom: env(safe-area-inset-bottom)` shrinks the inner container's content area on devices with home indicators (~34px inset). Implementation matches spec code sample exactly; needs real-device testing to confirm visual impact. Fix: change `h-16` to `min-h-16` or move safe-area padding to the outer `<div>`.
- Zustand rehydration layout flash — between mount and Zustand persist rehydration, `sidebarCollapsed` holds default value (false), causing a brief sidebar expand→collapse animation for users who had the sidebar collapsed. Pre-existing pattern from Story 3.3.
- Hardcoded `unreadCount={3}` in TopBar `<NotificationsBell>` — placeholder value from Story 3.3. Will be replaced with real notification data when notification system is implemented (Story 3.11).
- Hardcoded user objects in layout files (`"Demo User"`, `"Admin User"`) — placeholder from Story 3.3. Auth context integration in Story 3.8/3.12.
- Language selector hardcoded to "BG" with no click handler — placeholder from Story 3.3. i18n implementation in Story 3.7.

## Deferred from: code review of story 3-6-react-hook-form-zod-validation-patterns (2026-04-08)

- `useFormField` guard check ordering in `form.tsx:47-53` — `getFieldState(fieldContext.name)` is called before the null check, and the check itself is unreachable because `useContext` returns `{}` (truthy default). Pre-existing shadcn pattern; harmless in practice since `useFormField` is always called inside `FormFieldInternal` which provides the context.
- `max-h-10` (40px) in FormMessage clips multi-line error messages — `overflow-hidden` with `max-h-10` will truncate Zod error text longer than one line. Standard shadcn pattern; multi-line validation errors are uncommon in practice. Consider `max-h-20` if long error messages become necessary.
- Unsafe `as React.ReactElement` cast on children render-prop return in `FormField.tsx:113` — if `children()` returns `null`, `undefined`, or a `Fragment`, the cast is incorrect and Radix `Slot` may fail at runtime. TypeScript enforces the contract at call sites, limiting real-world impact.
- `String(error?.message)` in `form.tsx:147` renders literal string "undefined" when a `FieldError` object has no `.message` property — `String(undefined)` evaluates to `"undefined"`. Pre-existing shadcn pattern; unreachable with zodResolver (which always populates message), only triggered by manual `setError()` without message argument.
- Radix Select crashes on empty-string option value in `FormField.tsx:121-137` — Radix UI Select v2 throws at runtime if any `<SelectItem>` has `value=""`. No guard on the `options` prop rejects empty strings. Pre-existing Radix UI constraint; caller responsibility to provide valid option values.
- File input `field.onChange` captures fakepath string instead of FileList in `FormField.tsx:169-179` — RHF's internal `getEventValue()` extracts `event.target.value` (browser fakepath like `C:\fakepath\file.pdf`) rather than `event.target.files`. Built-in `type="file"` variant is intentionally basic per spec; advanced file handling uses render-prop children (S03.09 logo upload).

## Deferred from: code review of story 3-9-company-profile-setup-wizard (2026-04-08)

- Auth hydration race in setup page — authenticated users visiting `/setup` directly may flash-redirect to `/login` before Zustand auth store hydrates. Pre-existing pattern across all protected pages; full route guard enforcement scoped to S03.12 AuthGuard. Source: S03.09 edge-case review.
- Multi-tab wizard sync — completing the wizard in Tab A does not invalidate Tab B's in-memory Zustand state. Tab B can re-submit the wizard against an already-configured profile. Zustand persist does not cross-tab sync by default. Add `window.addEventListener('storage', ...)` or check server-side profile completion status on mount when production multi-device usage patterns emerge. Source: S03.09 edge-case review.
- `user?.companyId ?? "stub-company-id"` fallback in `handleComplete` — when the real API replaces the stub, this hardcoded fallback will silently send an invalid company ID. Remove the fallback and guard with `if (!user?.companyId) { setSubmitError(...); return; }` at E02 integration. Source: S03.09 blind review.
- Case-sensitive email deduplication in Step 4 `addEmail` — `"JOHN@example.com"` and `"john@example.com"` are treated as distinct entries. Emails are case-insensitive per RFC 5321 (domain) and conventionally case-insensitive (local part). Normalize with `.toLowerCase()` before comparison. Low-severity; defer to production hardening. Source: S03.09 blind+edge review.
- Next.js 15 `params` async breaking change — setup page and register page use synchronous destructuring `{ params: { locale } }` which will fail at runtime in Next.js 15+ where `params` is an async `Promise`. Codebase-wide migration concern; not specific to S03.09. Source: S03.09 edge-case review.

## Deferred from: code review of story 3-12-client-side-route-guards-auth-redirects (2026-04-09)

- Middleware cookie value not validated — `client middleware.ts:56`, `admin middleware.ts:54`. `req.cookies.get("eusolicit-session")` checks presence only; any non-empty string passes. Documented as intentional per spec ("signature validation happens server-side in E02 endpoints"). Middleware is a secondary guard; client-side AuthGuard is primary. Source: S03.12 edge-case review.
- `logout()` doesn't invalidate server-side session or clear `eusolicit-session` cookie — `client (protected)/layout.tsx:80`, `admin (protected)/layout.tsx:74`. Only clears Zustand state and localStorage. The `eusolicit-session` cookie is httpOnly (set by E02 backend), so client-side code cannot clear it. Requires a `/auth/logout` API endpoint — E02 backend scope. Source: S03.12 blind+edge review.
- `loginUser()` API response shape not validated with schema — `client login/page.tsx:63-64`. Response from `loginUser()` is consumed without Zod validation before calling `login(response.user, response.token, response.refreshToken)`. Pre-existing pattern from S3.8. Source: S03.12 blind review.
- `setTokens()` action creates partial auth state (`user=null, isAuthenticated=true`) — `auth-store.ts:41-42`. If `setTokens` is called without a preceding `login` or `setUser`, the guard condition `isAuthenticated && token` passes but downstream components reading `user.name` / `user.email` would NPE. Pre-existing S3.5 store design. Source: S03.12 edge-case review.
- `handleLocaleChange` uses naive `String.replace()` for locale segment swap — both protected layouts. `pathname.replace(\`/${locale}\`, \`/${newLocale}\`)` replaces only the first occurrence. If a path segment coincidentally contains the locale code (e.g. `/bg/settings/bg-region`), the result is correct for the first occurrence but fragile. Pre-existing from S3.3/S3.7. Source: S03.12 edge-case review.

## Deferred from: code re-review of story 3-12-client-side-route-guards-auth-redirects (2026-04-09)

- No hydration timeout for corrupt localStorage — `AuthGuard.tsx:33-49`. If localStorage contains malformed JSON under key `eusolicit-auth-store`, Zustand persist may fail to hydrate and `onFinishHydration` never fires. User sees permanent spinner with no recovery path except clearing browser data. Defensive improvement: add a 5-second timeout that treats hydration failure as unauthenticated. Source: S03.12 edge-case re-review.
- AuthRedirect renders children during redirect (brief flash of auth page) — `AuthRedirect.tsx:63`. Authenticated users visiting `/login` briefly see the login form before the redirect effect fires. The spec describes AuthRedirect as a "lighter" component; this is cosmetic, not a security concern. UX improvement: gate rendering behind hydration state like AuthGuard does. Source: S03.12 blind+edge re-review.
- Cross-app localStorage key collision — `auth-store.ts:47`. Both client and admin apps use the identical Zustand persist key `eusolicit-auth-store`. Not exploitable when apps run on different origins (ports), but if deployed behind a reverse proxy on the same origin (e.g., `/client/` and `/admin/`), a regular user's client auth state leaks into the admin app. Pre-existing S3.5 design. Namespace keys per app if deployment model changes. Source: S03.12 edge-case re-review.
- Token expiry not tracked client-side — `AuthGuard.tsx:54`. Guard checks `isAuthenticated && token` but never validates JWT expiry. A user returning after token expiry sees the protected UI until an API call triggers the refresh interceptor, which on failure calls `logout()`. Window of stale UI with broken data fetching. Architectural decision: API endpoints validate JWTs server-side. Proactive expiry check would improve UX. Source: S03.12 edge-case re-review.

## Deferred from: code review of story 11-1-espd-profile-compliance-framework-db-schema-migrations (2026-04-09)

- Redundant unique constraint + unique index on `admin.platform_settings.key` — `platform_settings.py:17-19`, `002_compliance_framework_schema.py:126-135`. The ORM model and migration both declare a `UniqueConstraint("key", name="uq_platform_settings_key")` AND a separate unique `Index("ix_platform_settings_key", "key", unique=True)`. PostgreSQL implements UNIQUE constraints via unique indexes internally, creating two functionally identical B-tree indexes on `key`. Both are required by spec (AC3 specifies `key UNIQUE`, AC4 specifies `ix_platform_settings_key (unique)`), so this is a spec-level redundancy. Could cause `alembic check` to flag duplication. Consider consolidating in a future spec revision. Source: S11.01 blind+auditor review.

## Deferred from: code review of story 11-8-compliance-framework-crud-api-admin (2026-04-09)

- JWT secret cached forever with no rotation mechanism — `config.py:32`. `@lru_cache` on `get_settings()` freezes the JWT secret for the process lifetime. No production-time secret rotation without restart. Source: S11.08 blind review.
- HS256 symmetric signing weak for multi-service token verification — `core/security.py`. Every service holds the same signing secret; compromise of any one allows forging tokens for all. Consider RS256/ES256 when auth infrastructure matures. Source: S11.08 blind review.
- No audience/issuer claim validation on JWT decode — `core/security.py:60`. `jwt.decode()` only validates `algorithms`. A token minted for a different service sharing the same secret would be accepted. Add `audience` and `issuer` claims. Source: S11.08 blind review.
- Race condition in lazy singleton engine initialization — `dependencies.py:27-37`. `get_engine()` uses `global _engine` without an async lock. Concurrent requests at startup can create multiple engines/session factories. Source: S11.08 blind+edge review.
- Concurrent delete+update race can cause StaleDataError 500 — `services/compliance_framework_service.py`. Simultaneous delete and update on the same framework can produce `StaleDataError` or silent no-op. Consider `SELECT ... FOR UPDATE` for write paths. Source: S11.08 edge review.
- `created_at`/`updated_at` nullable in DB but required in response schema — `models/compliance_framework.py:73-83`. Migration 002 created these columns without NOT NULL. If a row has NULL timestamps (e.g., from direct SQL), `ComplianceFrameworkResponse` serialization raises ValidationError 500. Source: S11.08 edge review.
- Auto-commit on read-only GET endpoints — `dependencies.py:47`. `get_db_session` always calls `session.commit()` after yield, even for read-only GET endpoints. Wasteful and can cause subtle issues with implicit flushes. Source: S11.08 blind review.
- CASCADE FK on `opportunity_compliance_frameworks` contradicts deletion guard intent — `models/opportunity_compliance_framework.py:33`. `ondelete="CASCADE"` means if the app-level guard is bypassed, the DB cascades deletes rather than blocking. Semantically contradicts the 409-guard intent of AC7/AC8. Source: S11.08 auditor review.
- `rule_id` field accepts arbitrary unbounded strings — `schemas/compliance_framework.py:20`. No `min_length`, `max_length`, or pattern constraint on inbound `rule_id`. Oversized or malformed values persist into JSONB. Source: S11.08 edge review.
- HTTPException triggers unnecessary rollback in `get_db_session` — `dependencies.py:46-51`. Application-level errors (404/409) caught by `except Exception`, triggering pointless ROLLBACK. If rollback itself fails, the original error is masked. Source: S11.08 edge review.
