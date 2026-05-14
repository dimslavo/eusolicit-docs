# Story 4.28: Tier-to-SirmaAI Rate-Limit Sync (NFR-25)

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend engineer landing PRD-amendment NFR-25 ("EU Solicit tier upgrades and downgrades shall be mirrored to the SirmaAI Organization rate-limit policy via `PATCH /api/admin/organizations/{orgId}/rate-limit` within 60 seconds of the tier change event. Default mappings: Free → 100 agent-runs/day, Starter → 1k/day, Professional → 10k/day, Enterprise → 100k/day; Pro+ 20k/day from Epic 15") + Epic 4 amendment AC 12 ("Tier-to-rate-limit sync (NFR-25) reflects every `subscription.changed` event on SirmaAI within 60s p95") + Implementation-Readiness Concern #1 (NFR-25 unowned — this story is the explicit owner) + E24 AC line 30 ("Free-tier rate limit (100 runs/day per NFR-25) is applied automatically by E04 amendment S04.28 tier-sync flow on company create — no tier-conditional code path in E24 itself")**,

I want **(a) a new persistent asyncio background-task consumer `sirmaai_rate_limit_sync_consumer` in `sirmaai-gateway` (NOT in client-api — the gateway is the single integration point with SirmaAI per architecture amendment §3.1, and the existing client-api `tier_cache_consumer.py` from Story 8.14 stays scoped to its purpose, namely deleting the local `tier:{company_id}` Redis cache key; this story adds a SECOND consumer group on the SAME `subscription.changed` Redis Stream so the two consumers run in parallel without competing for messages — Redis Streams consumer-group semantics: each group sees every message exactly once, so the existing `client-api:tier-cache-invalidator` group continues to invalidate the local cache while the new `sirmaai-gateway:rate-limit-sync` group calls SirmaAI); (b) a YAML-driven tier→rate-limit mapping at `services/sirmaai-gateway/config/tier_rate_limits.yaml` (NEW directory + NEW file — per Epic 4 amendment line 510: "Tier-mapping table maintained in `services/sirmaai-gateway/config/tier_rate_limits.yaml` for commercial calibration without code changes"), schema-validated on load via a Pydantic `TierRateLimitMap` model, defaults baked in per NFR-25 + Epic 15 Pro+ row: `free=100/d, starter=1000/d, professional=10000/d, pro_plus=20000/d, enterprise=100000/d`; each tier row carries all four required SirmaAI `RateLimitConfigRequestDto` fields (`requestsPerMinute`, `requestsPerHour`, `requestsPerDay`, `burstCapacity`) because the SirmaAI API requires the full DTO on every PUT — daily targets are translated to the other dimensions in the YAML itself (NOT computed in code; the operator-facing tuning surface MUST be the daily ceiling and burst, not a programmatic derivation that's invisible to commercial review); (c) a new `SirmaAIRateLimitClient` in `sirmaai_gateway.services.sirmaai_rate_limit_client` modeled exactly on the existing `SirmaAIKeyManagementClient` pattern (admin Bearer header via `SIRMAAI_ADMIN_API_KEY`, lifespan-managed httpx.AsyncClient from `kraftdata_client.get_client()`, explicit `httpx.Timeout(connect=10.0, read=30.0, write=10.0, pool=5.0)`, wrapped in `circuit_breaker(retry(_attempt))` using the SAME two-layer resilience composition as S04.22/S04.23/S04.24, single circuit-breaker key `"sirmaai_rate_limit_admin"` shared across all rate-limit ops — a SirmaAI admin-API outage trips ONE breaker, not 5,000 per-tenant breakers); (d) the consumer reads each `subscription.changed` event, parses the EventPublisher envelope (top-level `company_id` fallback for unit-test mocks, then nested `payload` JSON for production envelopes — mirroring the existing `tier_cache_consumer.py` parse pattern verbatim so a future event-format change touches BOTH consumers symmetrically), resolves `company_id → (sirmaai_org_id, sirmaai_project_id, current_tier)` via the existing `SirmaAIProject` ORM model joined with `Subscription` (E08 canonical table — the `subscription.changed` event payload echoes `new_tier`, but the consumer reads from DB as the source of truth — defensive against stale events; tie-breaker: if event `new_tier` ≠ DB `subscription.tier`, log WARN `tier_mismatch` and use DB value, because the DB has the post-commit truth); (e) calls `PATCH /api/admin/organizations/{orgId}/rate-limit` *NOTE: the SirmaAI OpenAPI v3 actually defines this as `PUT /api/admin/organizations/{organizationId}/rate-limit` — the epic line 510 PRD-amendment wording says "PATCH" but the upstream spec is PUT. Use **PUT** to match SirmaAI's contract; the NFR-25 intent is unchanged (mirror the limit), and the verb discrepancy is recorded as a known-divergence note in the Senior Developer Review section of this story.* with `RateLimitConfigRequestDto` body assembled from the YAML row matching the resolved tier; (f) emits an audit row to `gateway.rate_limit_sync_audit` (NEW table — Alembic migration 006 — captures `(company_id, sirmaai_org_id, old_tier, new_tier, requested_dto JSONB, response_status, response_body_excerpt, error_type, latency_ms, synced_at)`; serves as the operator-facing forensic trail when a customer disputes a rate-limit decision — "did the sync actually run, did SirmaAI 200 it, what dimensions were requested" — without it, the only forensic surface is structlog which is not queryable per-tenant); (g) idempotent: the consumer is safe to re-process the same `subscription.changed` event N times (Redis Streams retry semantics) because a PUT against SirmaAI is a full-replace operation — the second PUT lands the same dimensions, no drift; the audit row writes once per event (the consumer's existing `EventConsumer.ack` handshake handles dedup at the message-id layer); (h) DLQ on `_DLQ_MAX_RETRIES = 3` — mirroring the existing tier_cache_consumer DLQ threshold; failures after 3 retries are logged at ERROR + audited with `error_type` populated; the event is NOT re-tried beyond that (the 5-min reconciler-equivalent for rate-limit drift is the S04.28 follow-up nightly check OUT OF SCOPE for this story — explicitly deferred as backlog item E04-R-007); (i) flag-aware: `SIRMAAI_GATEWAY_ENABLED=false` short-circuits the consumer at startup AND at every loop iteration — the FastAPI lifespan still creates the asyncio task (so toggling the flag without restart works in the runtime direction `false→true`), but the loop body returns `flag_off` early on each iteration (defence-in-depth on top of lifespan-level skip, matching the rotate_keys.py pattern at line 60 — re-using the discipline so a flag toggle never produces a half-active state); (j) **NO** changes to `tier_cache_consumer.py` — that consumer remains the local-cache invalidator and is not refactored; (k) **NO** changes to the `subscription.changed` event publishers (`webhook_service._publish_subscription_changed` + `billing_service.provision_new_company_billing_bg`) — the payload schema is preserved verbatim (adding fields to a Redis Streams event is a separate compatibility concern owned by E28); (l) **NO** new Celery task or Beat schedule — this is a Redis Streams asyncio consumer (request/event-driven), not a Celery scheduled job (cadence-driven); rationale: the 60-second SLO is met by event-driven processing (single-digit-ms intra-broker delay + ~1-3s PUT call), whereas a 5-min Celery sweep would already burn 5min/600s = 83% of the SLO budget for messages arriving immediately after a tick (l') reconciler-of-drift IS deferred (item E04-R-007) — this story does not own retroactive correction of tenants whose rate-limit drifted while sirmaai-gateway was down; the reconciler-of-drift backlog story will scan `gateway.rate_limit_sync_audit` for tenants whose last `synced_at` < now() - 24h or whose last response_status >= 400 and replay the sync; (m) **NO** new public-facing HTTP endpoint — the consumer is internal; an admin-replay surface ("force re-sync for company X") is deferred to E28 alongside the drift reconciler; (n) **cross-tenant negative test invariant** (project memory rule — non-negotiable): an integration test seeds two companies (PROJ_A on `professional`, PROJ_B on `starter`), publishes a single `subscription.changed` event with `company_id=PROJ_A, new_tier=enterprise`, asserts EXACTLY ONE outbound PUT was recorded by respx to `/api/admin/organizations/{PROJ_A.sirmaai_org_id}/rate-limit` with `requestsPerDay=100000`, and asserts ZERO outbound PUTs were recorded for `PROJ_B.sirmaai_org_id` — proves the resolution path is per-event-tenant and never leaks PROJ_A's tier event into PROJ_B's organization; (o) **timeout test** asserts every outbound PUT carries `httpx.Timeout(connect=10.0, read=30.0, …)` — same discipline as `sirmaai_key_client.py` line 67 (delivery rules §Security defaults: every httpx call has an explicit timeout); (p) **secret discipline**: NEVER log `SIRMAAI_ADMIN_API_KEY` at any level; PUT response bodies are truncated to 500 chars and recorded in the audit row's `response_body_excerpt` column ONLY (not logged) — the body can echo internal organization metadata; (q) **schema isolation invariant** (delivery rules §Database & schema rules + ADR-001): the new `gateway.rate_limit_sync_audit` table goes in the `gateway` schema (sirmaai-gateway owns the write); the consumer reads `client.sirmaai_projects` and `client.subscriptions` via async SQLAlchemy with the SAME ai_gateway_role grant pattern already used by `cache_invalidation_consumer.py` (which reads `client.sirmaai_projects` for cache invalidation per S04.21 init grants) — the grant precedent already exists, this story does NOT introduce a new cross-schema-read pattern; (r) the existing `cache_invalidation_consumer.py` in sirmaai-gateway is a CONSUMER PATTERN reference but NOT the same Redis Stream — it listens on `sirmaai.key_rotated` (S04.22 internal event), not `subscription.changed` — DO NOT collapse the two consumers into one (separation of concerns: key rotation invalidates the project_cache; subscription change syncs the rate limit; conflating them couples two unrelated lifecycles)**,

so that **(1) PRD NFR-25 — the only uncovered NFR in the SirmaAI amendment readiness gate per Implementation-Readiness Concern #1 — gets its owning implementation and the readiness gate flips from ❌ GAP to ✅ Covered for NFR-25; (2) a customer paying for Enterprise gets their 100k/day rate-limit reflected on SirmaAI within 60 seconds of the Stripe webhook firing — the SLO that lets Sales tell prospects "your tier upgrade is live in under a minute"; (3) a customer downgrading from Pro+ to Starter has their rate-limit tightened to 1k/day within 60 seconds — closes the **abuse window** where a churning customer could continue burning Pro+ throughput while the platform thinks they're on Starter (commercial-loss prevention); (4) the operator-facing tier-mapping file `tier_rate_limits.yaml` lets Finance/Commercial calibrate per-tier rate-limits without a code deploy — the file is the contract between Product and Engineering for "what does each tier cost in agent-runs"; (5) the `gateway.rate_limit_sync_audit` table answers "why does Company X have rate-limit Y on SirmaAI" with a single-row SELECT during dispute resolution — replaces the unauditable status quo where the only signal is "did the structlog event fire 3 weeks ago"; (6) the E24 Free-tier provisioning flow gets its rate-limit applied for free — Free-tier company creation publishes `subscription.changed{old_tier=null, new_tier=free}` (per `provision_new_company_billing_bg` line 808: it currently publishes `old_tier="free", new_tier="professional"` for the trial-provision case; the Free-tier-default-on-trial-expiry case at billing_service handles tier→`free` transitions), this consumer picks it up, calls SirmaAI with 100/day — no tier-conditional code lives in E24 (Deb decision 2026-05-12, captured in E24 AC line 30); (7) the §11.3 risk #2 ("SirmaAI as second critical external dependency") gets its **rate-limit-side observability** — operators monitoring the gateway can grep `rate_limit_sync.failed` to detect SirmaAI-side admin-API outages independently of agent-run failures; (8) the `circuit_breaker(retry(http_factory))` two-layer resilience contract gets one more well-bounded consumer that adds a 5,000th call to an existing tested code path rather than inventing a third resilience composition, keeping the gateway's circuit-breaker registry small and the operator's mental model clean**.

## Acceptance Criteria

1. **YAML tier→rate-limit mapping at `services/sirmaai-gateway/config/tier_rate_limits.yaml`** (NEW directory + NEW file):

   ```yaml
   # services/sirmaai-gateway/config/tier_rate_limits.yaml
   #
   # SirmaAI Organization rate-limit defaults per EU Solicit subscription tier.
   # Sourced from PRD NFR-25 + Epic 15 (Pro+ tier). Operator-tunable for commercial
   # calibration WITHOUT a code deploy.
   #
   # Tier MUST exist in eusolicit_models.enums.SubscriptionTier (free, starter,
   # professional, pro_plus, enterprise). Unknown tiers fail schema validation
   # on consumer startup (fail-fast — never silently fall back).
   #
   # All four fields are required by SirmaAI RateLimitConfigRequestDto:
   #   requestsPerMinute, requestsPerHour, requestsPerDay, burstCapacity
   #
   # Daily targets come from NFR-25 directly; minute/hour/burst are commercial
   # smoothing defaults — Engineering owns the daily floor, Commercial owns the
   # other three. Burst is set to ~10% of the minute budget rounded up.
   tiers:
     free:
       requestsPerMinute: 5
       requestsPerHour: 50
       requestsPerDay: 100
       burstCapacity: 10
     starter:
       requestsPerMinute: 20
       requestsPerHour: 250
       requestsPerDay: 1000
       burstCapacity: 30
     professional:
       requestsPerMinute: 100
       requestsPerHour: 1500
       requestsPerDay: 10000
       burstCapacity: 150
     pro_plus:
       requestsPerMinute: 150
       requestsPerHour: 2500
       requestsPerDay: 20000
       burstCapacity: 200
     enterprise:
       requestsPerMinute: 500
       requestsPerHour: 10000
       requestsPerDay: 100000
       burstCapacity: 750
   ```

   - File path is exactly `services/sirmaai-gateway/config/tier_rate_limits.yaml` (relative to the service root, matching the existing `agents_yaml_path` precedent on the deprecated `config/agents.yaml`).
   - The path is exposed as a config setting on `SirmaAIGatewaySettings`: `tier_rate_limits_yaml_path: str = "config/tier_rate_limits.yaml"` (env: `TIER_RATE_LIMITS_YAML_PATH`). Default is the in-tree path; production deployments can override to mount an operator-tuned file.
   - Loader lives in `services/sirmaai-gateway/src/sirmaai_gateway/services/tier_rate_limits.py` (NEW module) and exposes:
     - `class RateLimitDto(BaseModel)` with `requestsPerMinute: int, requestsPerHour: int, requestsPerDay: int, burstCapacity: int` (all `ge=1`, `model_config=ConfigDict(frozen=True)`).
     - `class TierRateLimitMap(BaseModel)` with `tiers: dict[SubscriptionTier, RateLimitDto]` (frozen). Uses the existing `eusolicit_models.enums.SubscriptionTier` StrEnum — DO NOT re-define the enum locally (S04.06 reuse discipline).
     - `def load_tier_rate_limits(path: str) -> TierRateLimitMap` — reads YAML, validates via Pydantic, raises `TierRateLimitsConfigError` on any of: file not found, YAML parse error, missing/unknown tier, missing/zero/negative field. The error message includes the offending row but NEVER the file contents.
     - `def get_tier_rate_limits() -> TierRateLimitMap` — lru_cache(maxsize=1) singleton accessor (mirrors `get_settings()` from `config.py`); tests use `get_tier_rate_limits.cache_clear()` to reset between cases.
   - **All 5 tier rows MUST be present**: `free`, `starter`, `professional`, `pro_plus`, `enterprise`. Missing one fails startup. Adding a sixth tier requires both an enum update AND a YAML update — the loader cross-checks the enum.

2. **New `SirmaAIRateLimitClient` in `services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_rate_limit_client.py`** (NEW module):

   - Class signature:
     ```python
     class SirmaAIRateLimitClient:
         def __init__(self, *, admin_api_key: str) -> None:
             self._admin_api_key = admin_api_key  # private, never reprable

         async def update_rate_limit(
             self,
             organization_id: str,
             dto: RateLimitDto,
         ) -> RateLimitConfigResponse:
             ...
     ```
   - Calls `PUT /api/admin/organizations/{organization_id}/rate-limit` with JSON body `dto.model_dump()`. **The verb is PUT** — see `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json` path `/api/admin/organizations/{organizationId}/rate-limit` which defines `put` (not `patch`). The Epic 4 amendment line 510 references "PATCH" verbatim from the PRD-amendment NFR-25 wording, but the upstream contract is **PUT**. Use PUT and record the divergence in the Senior Developer Review section.
   - Header `Authorization: Bearer {self._admin_api_key}` — the SAME admin token used by `SirmaAIKeyManagementClient` (config setting `sirmaai_admin_api_key`). NEVER log this value.
   - Header `Content-Type: application/json`.
   - **Explicit `httpx.Timeout(connect=10.0, read=30.0, write=10.0, pool=5.0)`** — copied verbatim from `sirmaai_key_client._KEY_MGMT_TIMEOUT` (admin-API tier latency profile, distinct from agent-run profile).
   - Wrapped in `circuit_breaker(retry(_attempt))` exactly like `sirmaai_key_client.create_api_key`:
     - Circuit-breaker key: `"sirmaai_rate_limit_admin"` (singleton — one breaker for all rate-limit ops, NOT per-tenant; a SirmaAI admin-API outage trips one circuit, not 5,000).
     - Threshold + cooldown read from `settings.circuit_breaker_threshold` / `settings.circuit_breaker_cooldown`.
     - Retry policy: inherited from `sirmaai_gateway.services.retry.with_retry` — 3 retries with exponential backoff on 5xx / timeout / connection errors.
   - HTTP error mapping (identical to `sirmaai_key_client`):
     - `httpx.TimeoutException` → `KraftDataTimeoutError` (retryable).
     - `httpx.ConnectError` → `KraftDataConnectionError` (retryable).
     - 5xx → `KraftDataAPIError(status, body[:500])` (retryable).
     - 4xx → `KraftDataAPIError(status, body[:500])` (**NOT** retryable — 400 = invalid DTO, 403 = admin-key revoked, 404 = unknown org — all of these need a human, not a retry).
     - `CircuitOpenError` (raised by circuit on cooldown) propagates to the consumer for ERROR-level logging + audit row.
   - Returns a typed `RateLimitConfigResponse` Pydantic model (id, organizationId, requestsPerMinute, requestsPerHour, requestsPerDay, burstCapacity, isActive, created, updated) — mirrors SirmaAI's `RateLimitConfigResponseDto`. Used downstream to populate the `response_status`/`response_body_excerpt` audit columns.
   - **Body logging**: response bodies are NEVER logged at any level — only `status_code` and `error_type` (the body can echo organization metadata or 5xx exception traces with bearer-in-payload echoes — mirror the `S04.22 M9` lesson + the `sirmaai_key_client.py` lines 583-585 discipline of slicing `resp.text[:200]` and the comment "limited to avoid echoing bearer tokens in 5xx bodies").

3. **New persistent asyncio consumer in `services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`** (NEW module):

   - Stream: `subscription.changed` (constant `_STREAM`).
   - Consumer group: `sirmaai-gateway:rate-limit-sync` (constant `_GROUP` — distinct from the existing `client-api:tier-cache-invalidator` group; Redis Streams consumer-group semantics deliver each message to both groups independently, so no contention).
   - Consumer name: `sirmaai-gateway-sync-1` (constant `_CONSUMER`).
   - Block-ms / count / DLQ_MAX_RETRIES: re-use `_BLOCK_MS=2000`, `_COUNT=10`, `_DLQ_MAX_RETRIES=3` — same constants as `client_api.services.tier_cache_consumer` (consistency across consumers reduces operator surprise).
   - Entry point: `async def run_rate_limit_sync_consumer(app: FastAPI) -> None` — the same signature as `run_tier_cache_invalidator` so it plugs into the existing lifespan loop without invention.
   - Top-of-loop flag check: `if not settings.sirmaai_gateway_enabled: await asyncio.sleep(60); continue` — the consumer survives a flag-off-at-boot (so a flag flip without restart works in the `false→true` direction) but skips all message processing when off. Mirrors the rotate_keys.py defence-in-depth (it's NOT enough to skip the lifespan create_task call — a flag toggle mid-run must produce no-op iterations, not crashes).
   - Event-envelope parsing — mirror `tier_cache_consumer.py` lines 80-100 verbatim (top-level `company_id` fallback for unit-test mocks; nested `payload` JSON for production envelopes). Add ONE additional read: `new_tier` from either top-level or `payload.new_tier`. If `new_tier` is missing, log WARN `missing_new_tier`, ack, skip.
   - Resolution: open an async SQLAlchemy session via `get_session_factory()` (S04.22 pattern). Run a single SELECT joining `client.sirmaai_projects` (for `sirmaai_org_id`) with `client.subscriptions` (for current `tier` — source of truth). Acceptable forms:
     ```python
     stmt = (
         select(SirmaAIProject.sirmaai_org_id, Subscription.tier)
         .join(Subscription, Subscription.company_id == SirmaAIProject.company_id)
         .where(SirmaAIProject.company_id == company_id)
         .where(SirmaAIProject.provisioning_status == "provisioned")
     )
     ```
     - If the SELECT returns 0 rows: log INFO `no_sirmaai_project` (provisioning still pending, or company archived), ack, skip. Do NOT call SirmaAI.
     - If DB-`tier` ≠ event-`new_tier`: log WARN `tier_mismatch` with both values, use **DB value** as authoritative (post-commit truth — event might be stale).
   - Call `SirmaAIRateLimitClient.update_rate_limit(organization_id=sirmaai_org_id, dto=tier_rate_limits.tiers[db_tier])`.
   - Write audit row (see AC 4).
   - On success: log INFO `rate_limit_sync.synced` with `company_id`, `sirmaai_org_id`, `new_tier`, `requests_per_day`, `latency_ms`. Ack the message.
   - On `KraftDataAPIError` (4xx — non-retryable): log ERROR `rate_limit_sync.permanent_failure` with `status_code`, `error_type` (NEVER `error=str(exc)`). Write audit row with `error_type` set. Ack the message (no point retrying a 400/403/404).
   - On `KraftDataTimeoutError` / `KraftDataConnectionError` / `KraftDataAPIError` (5xx) / `CircuitOpenError`: log WARN `rate_limit_sync.transient_failure`. Write audit row with `error_type` set. **Do NOT ack** — let the EventConsumer's process_pending DLQ mechanism replay up to `_DLQ_MAX_RETRIES=3` times, then drop the message at ERROR level. Drift correction is deferred (E04-R-007 backlog item).
   - On any other Exception (defensive bottom-of-loop catch): log ERROR with `error_type` ONLY (never `error=str(exc)` per S04.22 M9). Ack the message (do NOT loop on a programmer bug). The bare-except-replacement uses an explicit `except Exception:  # noqa: BLE001` comment carrying forward the S04.22 H6 discipline.

4. **New `gateway.rate_limit_sync_audit` table — Alembic migration `006_rate_limit_sync_audit.py`**:

   - Schema (PostgreSQL DDL, executed via Alembic in the `gateway` schema):
     ```sql
     CREATE TABLE gateway.rate_limit_sync_audit (
         id              UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
         company_id      UUID         NOT NULL,
         sirmaai_org_id  TEXT         NOT NULL,
         old_tier        TEXT         NULL,           -- nullable for first-sync rows
         new_tier        TEXT         NOT NULL,
         requested_dto   JSONB        NOT NULL,       -- the full RateLimitConfigRequestDto sent
         response_status INTEGER      NULL,           -- HTTP status from SirmaAI; NULL on network/circuit-open failure
         response_body_excerpt TEXT   NULL,           -- truncated to 500 chars; never bearer-bearing
         error_type      TEXT         NULL,           -- type(exc).__name__ on failure; NULL on success
         latency_ms      INTEGER      NOT NULL,
         synced_at       TIMESTAMPTZ  NOT NULL DEFAULT now()
     );
     CREATE INDEX ix_rate_limit_sync_audit_company_id ON gateway.rate_limit_sync_audit(company_id);
     CREATE INDEX ix_rate_limit_sync_audit_synced_at  ON gateway.rate_limit_sync_audit(synced_at);
     CREATE INDEX ix_rate_limit_sync_audit_error_type ON gateway.rate_limit_sync_audit(error_type)
         WHERE error_type IS NOT NULL;  -- partial index — most rows are successes
     ```
   - **No cross-schema FKs** — `company_id` is plain UUID (not a FK to `client.companies`) because schema-isolation invariant ADR-001 forbids cross-schema FKs except the existing `client.companies` precedent in `client.sirmaai_projects.company_id` (S04.21). This audit table is owned by `gateway`; cleanup on company archive is handled by the existing tenant-soft-delete flow (E24 — outside this story).
   - Migration is reversible (downgrade drops the table + indexes).
   - **Role grant invariant**: `ai_gateway_role` already has default CRUD on `gateway.*` per `infra/postgres-init/*.sql` (S04.21 grants). NO new role grant SQL in the migration.
   - ORM model at `services/sirmaai-gateway/src/sirmaai_gateway/models/rate_limit_sync_audit.py` (NEW), mirroring the `WebhookLog` model pattern. Add to `models/__init__.py` re-exports.
   - **Cardinality**: one row per `subscription.changed` event processed (success or failure); a 3-retry DLQ produces up to 3 audit rows for the same `company_id` with the same `new_tier` but distinct `synced_at`. Operators querying the table use `ORDER BY synced_at DESC LIMIT 1 WHERE company_id = ?` to read "the latest sync outcome".

5. **Lifespan wiring in `services/sirmaai-gateway/src/sirmaai_gateway/main.py`**:

   - In the existing `lifespan(app)` async context manager, ALONGSIDE the existing `cache_invalidation_consumer` task (which listens on `sirmaai.key_rotated`), create a NEW persistent task:
     ```python
     rate_limit_sync_task = asyncio.create_task(run_rate_limit_sync_consumer(app))
     app.state._rate_limit_sync_task = rate_limit_sync_task  # mirror cache_invalidation_consumer state-attach pattern
     ```
   - On shutdown: `cancel()` the task and `await` it with a 5-second guard timeout (suppress `asyncio.CancelledError`, log WARN on `asyncio.TimeoutError`). Mirror the existing shutdown sequence for `cache_invalidation_consumer`.
   - **Flag-aware lifespan**: if `settings.sirmaai_gateway_enabled` is False at startup, STILL create the task — its loop-body flag check will produce no-ops. (Rationale per AC 3: tolerate a flag-flip-without-restart.)
   - The `tier_rate_limits.get_tier_rate_limits()` accessor is called once in `lifespan` startup to fail-fast on a malformed YAML — let the FastAPI process refuse to start rather than silently produce noop iterations forever.

6. **Cross-tenant negative test** (project memory rule — MANDATORY) at `services/sirmaai-gateway/tests/integration/test_rate_limit_sync_consumer.py`:

   - Provision two `client.sirmaai_projects` rows: `(company_id=COMPANY_A, sirmaai_org_id="ORG_A", provisioning_status="provisioned")` and `(company_id=COMPANY_B, sirmaai_org_id="ORG_B", provisioning_status="provisioned")`. Add matching `client.subscriptions` rows with `(COMPANY_A.tier=professional)` and `(COMPANY_B.tier=starter)`.
   - Mock SirmaAI via `respx`: register both `PUT https://stage.sirma.ai/api/admin/organizations/ORG_A/rate-limit` and `PUT https://stage.sirma.ai/api/admin/organizations/ORG_B/rate-limit` to return 200 with the canonical response DTO.
   - Publish ONE `subscription.changed` event with payload `{company_id: COMPANY_A, old_tier: professional, new_tier: enterprise, timestamp: …}` and update `COMPANY_A.tier=enterprise` in the DB BEFORE running the consumer step (mirrors production: webhook commits tier, then publishes event).
   - Run the consumer loop for one iteration (use the established `await asyncio.wait_for(consumer.consume(...), timeout=…)` pattern or directly call a `process_one_message` helper if extracted for testability).
   - **Assertions** (all four MUST pass):
     1. `respx.calls.call_count == 1`.
     2. The single recorded call's URL is `/api/admin/organizations/ORG_A/rate-limit` (NOT `ORG_B`).
     3. The recorded request body is `{"requestsPerMinute": 500, "requestsPerHour": 10000, "requestsPerDay": 100000, "burstCapacity": 750}` — the `enterprise` row from the YAML.
     4. Exactly ONE row in `gateway.rate_limit_sync_audit` with `(company_id=COMPANY_A, sirmaai_org_id="ORG_A", old_tier=professional, new_tier=enterprise, response_status=200, error_type IS NULL)`. ZERO rows for `COMPANY_B`.

7. **Tier-DB-mismatch test** at the same file:

   - Provision a `(company_id=COMPANY_C, tier=professional)` subscription.
   - Publish a `subscription.changed` event with `new_tier=enterprise` BUT do NOT update the DB (simulate a stale event — webhook published but DB rolled back, or a replay).
   - Assertions:
     1. The PUT body matches the `professional` YAML row (DB value wins).
     2. structlog records a WARN with `event="rate_limit_sync.tier_mismatch"`, `event_new_tier="enterprise"`, `db_new_tier="professional"`.
     3. The audit row has `new_tier=professional` (DB value).

8. **No-SirmaAI-project test** at the same file:

   - Publish a `subscription.changed` event with `company_id=COMPANY_NOPROJ` for which NO `client.sirmaai_projects` row exists (provisioning still pending — pre-S04.21-completion case).
   - Assertions:
     1. `respx.calls.call_count == 0` (no outbound call).
     2. structlog records an INFO with `event="rate_limit_sync.no_sirmaai_project"`.
     3. ZERO new rows in `gateway.rate_limit_sync_audit`.
     4. The Redis Streams message is ACK'd (no DLQ accumulation).

9. **Flag-off test** at the same file:

   - Patch `get_settings()` to return `sirmaai_gateway_enabled=False`.
   - Publish a `subscription.changed` event.
   - Assertions:
     1. `respx.calls.call_count == 0`.
     2. structlog records an INFO with `event="rate_limit_sync.flag_off"` (or the loop body simply early-returns without log — implementation choice; document in code which one was chosen).
     3. The consumer task is still running (no crash).

10. **Transient-failure DLQ test** at the same file:

    - Mock SirmaAI to return 500 on every call.
    - Publish a `subscription.changed` event.
    - Drive the consumer through 4 iterations (initial attempt + 3 retries — the `_DLQ_MAX_RETRIES=3` threshold).
    - Assertions:
      1. `respx.calls.call_count == 4` (1 + 3 retries, each call internally retried by `with_retry` 3 times — so up to 16 raw HTTP attempts; the test asserts the message-level retry count, not the HTTP-level retry count; use the `EventConsumer.process_pending` returned counter or a wrapping spy).
      2. `gateway.rate_limit_sync_audit` has 4 rows with `error_type="KraftDataAPIError"` and `response_status=500`.
      3. structlog records exactly 4 WARN `rate_limit_sync.transient_failure` events.
      4. The 5th iteration of the consumer loop does NOT re-process the message (DLQ threshold reached).

11. **Permanent-failure test** at the same file:

    - Mock SirmaAI to return 400 on the first attempt.
    - Publish a `subscription.changed` event.
    - Assertions:
      1. `respx.calls.call_count == 1` (no retries on 4xx).
      2. `gateway.rate_limit_sync_audit` has 1 row with `error_type="KraftDataAPIError"` and `response_status=400`.
      3. The Redis Streams message is ACK'd (permanent failure does not DLQ-loop).
      4. structlog records exactly ONE ERROR `rate_limit_sync.permanent_failure` with `status_code=400`, `error_type="KraftDataAPIError"`, **NO `error=str(exc)` field** (delivery rules §Code & logging conventions + S04.22 M9).

12. **Secret-leak audit test** at `services/sirmaai-gateway/tests/unit/test_rate_limit_client_secret_discipline.py`:

    - Construct a `SirmaAIRateLimitClient(admin_api_key="sk-deadbeef-token-must-not-leak")`.
    - Use `structlog.testing.capture_logs()` to capture all log records across one successful and one 5xx call (mock httpx via respx).
    - Use `re.search(r"sk-deadbeef-token-must-not-leak", json.dumps(captured_logs))` to assert ZERO matches across ALL log records.
    - Use the same regex on `repr(client)` and `str(client)` to assert the token never surfaces in object-printing paths.

13. **YAML-loader unit tests** at `services/sirmaai-gateway/tests/unit/test_tier_rate_limits.py`:

    - Happy path: load the in-tree `config/tier_rate_limits.yaml`, assert all 5 tiers parse, assert `free.requestsPerDay == 100` and `enterprise.requestsPerDay == 100000`.
    - Missing-tier negative: build a YAML missing `enterprise` → `load_tier_rate_limits()` raises `TierRateLimitsConfigError` with message naming `enterprise`.
    - Unknown-tier negative: build a YAML adding `vip: {…}` → raises `TierRateLimitsConfigError` (the loader cross-checks the `SubscriptionTier` enum).
    - Invalid-field negative: build a YAML with `free.requestsPerDay: -1` → raises (Pydantic `ge=1` constraint).
    - Malformed-YAML negative: pass a non-YAML string → raises with message NOT echoing the file contents.

14. **DoD checklist** (delivery rules §Definition of done — eusolicit commands):

    - `make lint` — green on all touched Python files.
    - `make type-check` — green (the new Pydantic models, the new client, the new consumer, the new migration ORM model all type-check under mypy).
    - `make test-service SVC=sirmaai-gateway` — green; the new unit + integration tests pass.
    - `make migrate-service SVC=sirmaai-gateway` from a clean `make reset-db` applies migration `006` without error; rollback (`alembic downgrade -1`) also succeeds.
    - `make test-integration` — green (the cross-tenant test joins postgres + redis).
    - `make coverage` — line coverage stays ≥ 80% on `sirmaai_gateway/services/sirmaai_rate_limit_client.py`, `sirmaai_gateway/services/rate_limit_sync_consumer.py`, `sirmaai_gateway/services/tier_rate_limits.py`, and the new ORM model.
    - **NO frontend changes** in this story — no `pnpm` commands required.
    - **NO E2E required** — the story is internal pipe-work; the end-to-end SirmaAI rate-limit enforcement is verified by SirmaAI itself on the next agent-run after the sync.

15. **Senior Developer Review section — record divergences**:

    - **Verb divergence**: PRD-amendment NFR-25 says `PATCH /api/admin/organizations/{orgId}/rate-limit`; SirmaAI OpenAPI v3 defines `PUT`. Implementation uses **PUT**. Record this in the story's "Known Deviations" subsection.
    - **`sirmaai_org_id` resolution**: the EU Solicit cluster currently uses a singleton org_id per `SirmaAIGatewaySettings.sirmaai_org_id` (architecture amendment Topology A, line 42 of arch amendment). However, `client.sirmaai_projects.sirmaai_org_id` is stored PER ROW (S04.21 schema, line 49 of `sirmaai_project.py`) for forward flexibility. This story reads `sirmaai_org_id` PER ROW from the `client.sirmaai_projects` join — so the implementation is correct regardless of whether the cluster moves to org-per-tenant later. Note this in dev notes.
    - **PUT body completeness**: SirmaAI requires all four DTO fields on every PUT (no partial updates per the OpenAPI `required` list). The YAML carries all four — Engineering owns daily floor + minute, Commercial owns hour + burst. Record this contract in the YAML file's header comment AND in dev notes.

## Tasks / Subtasks

- [x] **Backend: tier YAML + loader** (AC 1, AC 13)
  - [x] Create `services/sirmaai-gateway/config/tier_rate_limits.yaml` with the 5 tier rows as specified.
  - [x] Create `services/sirmaai-gateway/src/sirmaai_gateway/services/tier_rate_limits.py` with `RateLimitDto`, `TierRateLimitMap`, `load_tier_rate_limits()`, `get_tier_rate_limits()`, `TierRateLimitsConfigError`.
  - [x] Add `tier_rate_limits_yaml_path: str = "config/tier_rate_limits.yaml"` to `SirmaAIGatewaySettings`.
  - [x] Unit tests: happy path + 4 negative paths.

- [x] **Backend: SirmaAIRateLimitClient** (AC 2, AC 12)
  - [x] Create `services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_rate_limit_client.py` mirroring `sirmaai_key_client.py` structure.
  - [x] Define `RateLimitConfigResponse` Pydantic model.
  - [x] Wire `circuit_breaker(retry(_attempt))` with circuit key `"sirmaai_rate_limit_admin"`.
  - [x] Explicit 10s/30s timeouts; no body logging.
  - [x] Unit tests: 200 happy path, 5xx retry, 4xx no-retry, secret-leak audit.

- [x] **Backend: rate-limit-sync consumer** (AC 3, AC 5, AC 9, AC 10, AC 11)
  - [x] Create `services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py` mirroring `tier_cache_consumer.py` structure.
  - [x] Implement the resolution SELECT (sirmaai_projects JOIN subscriptions).
  - [x] Implement event-envelope parse (top-level + nested `payload` fallback).
  - [x] Implement DB-tier vs event-tier mismatch handling.
  - [x] Implement flag-aware loop body.
  - [x] Implement the 4xx-no-DLQ / 5xx-DLQ branching.
  - [x] Wire the task into `main.py` lifespan alongside `cache_invalidation_consumer`.

- [x] **Backend: audit table migration + ORM** (AC 4)
  - [x] Create Alembic migration `006_rate_limit_sync_audit.py` in `services/sirmaai-gateway/alembic/versions/`.
  - [x] Create `services/sirmaai-gateway/src/sirmaai_gateway/models/rate_limit_sync_audit.py`.
  - [x] Add to `models/__init__.py` re-exports.
  - [x] Migration 006 validated via integration test `run_migrations` fixture (Alembic upgrade head passes with 006 included; testcontainer-based validation replaces `make migrate-service` which requires client-api tables as a prerequisite).

- [x] **Cross-tenant + edge-case integration tests** (AC 6, AC 7, AC 8, AC 9, AC 10, AC 11)
  - [x] Create `services/sirmaai-gateway/tests/integration/test_rate_limit_sync_consumer.py` with all 7 scenarios (cross-tenant, tier-mismatch, no-project, flag-off, transient-DLQ, 4× DLQ, permanent-failure).

- [x] **Secret-discipline unit test** (AC 12)
  - [x] Create `services/sirmaai-gateway/tests/unit/test_rate_limit_client_secret_discipline.py`.

- [x] **DoD verification** (AC 14)
  - [x] `ruff check` — all 9 new/modified Python files pass.
  - [x] `mypy` — no new errors in sirmaai-gateway; existing 7 pre-existing errors unchanged.
  - [x] Unit tests: 261 passed / 0 failed in sirmaai-gateway unit suite.
  - [x] Integration tests: 7/7 new S04.28 tests pass; 46 other integration tests pass.
  - [x] Pre-existing failures (3 failures, 10 errors in other test files) are unchanged.

- [x] **Senior Developer Review divergence-recording** (AC 15)
  - [x] Known Deviations populated below.

## Dev Notes

### Architecture & invariants you MUST honor

- **PRD-amendment NFR-25** is the authoritative requirement; the daily ceilings are non-negotiable: Free=100, Starter=1k, Professional=10k, Pro+=20k (Epic 15 row), Enterprise=100k. The minute/hour/burst values in the YAML are commercial-tuning defaults — Engineering owns the daily floor.
- **Epic 4 amendment AC 12** — "within 60s p95" SLO. The event-driven asyncio consumer with `_BLOCK_MS=2000` typically sees latencies in single-digit-seconds end-to-end; the Celery-based alternative was rejected because a 5-min sweep alone burns 83% of the SLO budget.
- **Architecture amendment §3.1** — sirmaai-gateway is the single integration point with SirmaAI. The rate-limit consumer therefore lives in sirmaai-gateway, NOT in client-api (even though `subscription.changed` is published by client-api). The existing `client_api.services.tier_cache_consumer` stays unmodified.
- **ADR-004 (composite resilience)** — every outbound SirmaAI call wears the two-layer `circuit_breaker(retry(http_factory))` resilience. The new `SirmaAIRateLimitClient` is composed exactly the same way as `SirmaAIKeyManagementClient`; do NOT invent a third resilience pattern.
- **ADR-001 (schema isolation)** — `gateway.rate_limit_sync_audit` is owned by sirmaai-gateway and goes in the `gateway` schema. NO cross-schema FK to `client.companies` (plain UUID column). The cross-schema READS of `client.sirmaai_projects` and `client.subscriptions` are sanctioned by the existing `cache_invalidation_consumer.py` precedent (S04.21 init grants gave `ai_gateway_role` read access to `client.sirmaai_projects` for cache invalidation; this story extends that pattern to also read `client.subscriptions` — confirm the grant exists for `client.subscriptions` and add it to the init script if missing).
- **Cross-tenant negative test** is non-negotiable (project memory rule). AC 6 is the primary proof.
- **Secret discipline** — `SIRMAAI_ADMIN_API_KEY` NEVER appears in any log record at any level. `response.text` is sliced to `[:500]` and stored in `audit.response_body_excerpt` ONLY (not logged). Error type via `type(exc).__name__` ONLY (S04.22 M9).

### Reusable code paths from prior stories — DO NOT reinvent

- **`SirmaAIKeyManagementClient`** (`sirmaai_gateway/services/sirmaai_key_client.py`) — the pattern template for `SirmaAIRateLimitClient`. Copy the structure (admin-key constructor arg, circuit-breaker wrap, retry wrap, explicit timeout, typed exception mapping, response-body slicing). Do NOT subclass — the two clients are siblings, not parent-child.
- **`tier_cache_consumer.py`** (`client_api/services/tier_cache_consumer.py`) — the pattern template for `rate_limit_sync_consumer.py`. Copy the structure (asyncio.create_task lifespan attach, `EventConsumer` usage, top-level + nested-payload event parsing, DLQ via `process_pending`, asyncio.CancelledError re-raise discipline, 5-second back-off on loop_error). The new consumer is in `sirmaai-gateway`, not `client-api` — but the SHAPE is identical.
- **`cache_invalidation_consumer.py`** (`sirmaai_gateway/services/cache_invalidation_consumer.py`) — already-in-sirmaai-gateway example of a `subscription.changed`-style consumer (it consumes `sirmaai.key_rotated`, NOT `subscription.changed`, but the lifespan-attach + async loop pattern is the closest reference inside this service). Mirror its lifespan-attach and shutdown sequence in `main.py`.
- **`rotate_keys.py` + `rotate_webhook_secrets.py`** (`sirmaai_gateway/tasks/`) — the patterns for flag-off short-circuit + `error_type` discipline + Prometheus counter registration. The rate-limit sync does NOT use a Prometheus counter in this story (the audit table is the operator-facing surface; Prometheus surfacing is deferred to E28 alongside the drift reconciler).
- **`SirmaAIProject` ORM model** (`client_api/models/sirmaai_project.py`) — re-use the existing model definition; the sirmaai-gateway's `models/` directory should re-export it OR (cleaner) the consumer imports it directly from `client_api.models.sirmaai_project`. Verify the existing precedent — `cache_invalidation_consumer.py` already reads the row by raw SQL (no ORM), so following that precedent is also fine. Pick ONE pattern and document the choice in the consumer's module docstring.
- **`Subscription` ORM model** (`client_api/models/subscription.py`) — same situation. Raw SQL `SELECT tier FROM client.subscriptions WHERE company_id = …` is preferred to avoid pulling the full client_api ORM stack into sirmaai-gateway's import graph (the existing `cache_invalidation_consumer.py` uses raw SQL — follow that precedent).
- **`EventPublisher` / `EventConsumer`** (`eusolicit_common.events.publisher` / `eusolicit_common.events.consumer`) — re-use; do NOT introduce a second Redis-Streams abstraction.
- **`circuit_breaker` + `retry` + `kraftdata_client`** (`sirmaai_gateway/services/`) — re-use; do NOT spin up a new httpx pool.

### Files this story touches

| File | Action | Why |
|---|---|---|
| `services/sirmaai-gateway/config/tier_rate_limits.yaml` | NEW | Operator-tunable tier→rate-limit table. |
| `services/sirmaai-gateway/src/sirmaai_gateway/services/tier_rate_limits.py` | NEW | YAML loader + Pydantic schema + `get_tier_rate_limits()` accessor. |
| `services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_rate_limit_client.py` | NEW | `SirmaAIRateLimitClient` — PUT /api/admin/organizations/{id}/rate-limit wrapper. |
| `services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py` | NEW | Persistent asyncio consumer on `subscription.changed`. |
| `services/sirmaai-gateway/src/sirmaai_gateway/models/rate_limit_sync_audit.py` | NEW | SQLAlchemy ORM for `gateway.rate_limit_sync_audit`. |
| `services/sirmaai-gateway/src/sirmaai_gateway/models/__init__.py` | UPDATE | Re-export `RateLimitSyncAudit`. |
| `services/sirmaai-gateway/alembic/versions/006_rate_limit_sync_audit.py` | NEW | Alembic migration creating the audit table + 3 indexes. |
| `services/sirmaai-gateway/src/sirmaai_gateway/main.py` | UPDATE | Lifespan: create asyncio task for the new consumer; fail-fast YAML load. |
| `services/sirmaai-gateway/src/sirmaai_gateway/config.py` | UPDATE | Add `tier_rate_limits_yaml_path` setting. |
| `services/sirmaai-gateway/tests/unit/test_tier_rate_limits.py` | NEW | YAML loader happy + negative paths. |
| `services/sirmaai-gateway/tests/unit/test_rate_limit_client_secret_discipline.py` | NEW | Bearer-leak audit + 5xx/4xx error mapping unit tests. |
| `services/sirmaai-gateway/tests/integration/test_rate_limit_sync_consumer.py` | NEW | Cross-tenant + tier-mismatch + no-project + flag-off + DLQ + 4xx scenarios. |
| `infra/postgres-init/*.sql` | UPDATE (if needed) | Add `GRANT SELECT ON client.subscriptions TO ai_gateway_role` if not already present from S04.21. Verify before adding. |

### Test design notes (epic-04 test-design priority framework applied to S04.28)

The epic-level test design predates the SirmaAI amendment and does NOT enumerate S04.28 scenarios. Apply the P0/P1/P2 framework as follows:

- **P0** (critical, must pass in PR CI):
  - Cross-tenant integration test (AC 6) — proves per-tenant resolution path. THE project-memory invariant.
  - Permanent-failure test (AC 11) — proves 4xx does NOT DLQ-loop (a 400 from SirmaAI would otherwise wedge the consumer on a poison message indefinitely).
  - Secret-leak audit (AC 12) — proves `SIRMAAI_ADMIN_API_KEY` never surfaces in logs (regression guard on S04.22 M9).
  - Transient-failure DLQ test (AC 10) — proves the DLQ threshold is respected (a perpetual SirmaAI 500 outage would otherwise loop the audit table forever).

- **P1** (high — should pass in PR CI):
  - YAML happy + 4 negative paths (AC 13) — proves operator-facing config behaves predictably.
  - Tier-DB-mismatch test (AC 7) — proves DB is source-of-truth.
  - No-SirmaAI-project test (AC 8) — proves provisioning-pending case doesn't crash the consumer.
  - Migration `006` clean-apply + downgrade (AC 14).

- **P2** (medium — nice to have):
  - Flag-off behavior (AC 9) — defence-in-depth; the lifespan check covers most of the surface.
  - End-to-end timing assertion: from `EventPublisher.publish()` to `gateway.rate_limit_sync_audit.synced_at` < 60s p95. Skipped from PR CI because Redis Streams block-timing makes it flaky; tracked as a backlog observability item.

### Anti-patterns to avoid (S04.21 + S04.22 + S04.23 + S04.24 + S04.25 + S04.26 + S04.27 review lessons applied)

- **NEVER** put rate-limit logic in `client-api`. The single-integration-point invariant lives in sirmaai-gateway (architecture amendment §3.1). A future story refactoring this back into client-api would also have to refactor every other SirmaAI outbound call — don't start that drift here.
- **NEVER** modify `tier_cache_consumer.py`. The two consumers are intentionally distinct — same stream, different groups. Conflating them couples local-cache invalidation (a millisecond op) with SirmaAI admin-API calls (seconds, network-bound).
- **NEVER** call SirmaAI without the `circuit_breaker(retry(_attempt))` two-layer wrap (ADR-004). A bare `httpx.put(...)` would skip the cooldown that protects SirmaAI during outages.
- **NEVER** log `SIRMAAI_ADMIN_API_KEY` or any SirmaAI response body. The body can echo organization metadata and 5xx bodies can echo bearer tokens through error chains (S04.22 M9 lesson).
- **NEVER** use `error=str(exc)` anywhere — only `error_type=type(exc).__name__` (delivery rules §Code & logging conventions + S04.22 M9 + PR-time grep audit gate).
- **NEVER** retry on 4xx. The retry layer in `retry.py` already excludes 4xx; do NOT add a 4xx-retry branch in this story (would re-trigger S04.06 retry-discipline lessons).
- **NEVER** drop the cross-tenant negative test (AC 6). It is THE project-memory invariant. Removing it for "test simplification" is a non-starter.
- **NEVER** invent a Celery task for this. The event-driven asyncio consumer is the right primitive for an SLA-sensitive sync; a Celery sweep would burn the SLO budget on its own cadence.
- **NEVER** combine the YAML config with `agents.yaml` (which is retired post-S04.30). Tier-rate-limits is a new operator-facing surface and needs its own file; the agents.yaml retirement story is orthogonal.
- **NEVER** silently fall back when the YAML is missing a tier. Fail-fast on startup (AC 1 last bullet); a "default to free" fallback would silently throttle paying customers.
- **NEVER** add a Prometheus counter in this story. Audit table is the surface; metrics are deferred to E28 alongside the drift reconciler (E04-R-007 backlog).
- **NEVER** retry-loop on a poison message (4xx permanent failure). Ack on 4xx so the DLQ doesn't fill up with un-retryable failures (AC 11).

### Resolution path for ai_gateway_role grants on `client.subscriptions`

The existing init-script grants (`infra/postgres-init/*.sql`, S04.21) gave `ai_gateway_role` SELECT on `client.sirmaai_projects`. This story also reads `client.subscriptions`. **Verify this grant exists before adding new SQL** — `cache_invalidation_consumer.py` may already have warranted it, in which case the grant is present and no migration step is needed. If absent, add `GRANT SELECT ON client.subscriptions TO ai_gateway_role` to the canonical init script (NOT to the migration — grants live in the init scripts per project convention, alongside the schema-isolation invariant test in `tests/integration/test_db_schema_isolation.py`). If the canonical init scripts have the grant on a broader rule (e.g., `client.* read` for ai_gateway_role), do nothing.

### Known Deviations to record in the Senior Developer Review

1. **PUT vs PATCH verb**: PRD amendment NFR-25 wording says `PATCH`; SirmaAI OpenAPI v3 defines `PUT`. Implementation uses **PUT** (the upstream contract wins; the NFR-25 intent of "mirror the limit" is satisfied either way). Recommend updating the PRD amendment wording in a future doc-only patch — out of scope for this story.
2. **`sirmaai_org_id` per row vs cluster singleton**: the architecture amendment table at line 42 ("Topology A: One Org cluster-wide") suggests a singleton, but the `client.sirmaai_projects` schema (S04.21) stores `sirmaai_org_id` PER ROW. This story uses the per-row value — correct for both topology variants. Tag the future amendment if cluster migrates to org-per-tenant.
3. **All four DTO fields required on every PUT**: the SirmaAI `RateLimitConfigRequestDto` requires all of `requestsPerMinute`, `requestsPerHour`, `requestsPerDay`, `burstCapacity` (no partial updates). The YAML carries all four; the daily floor is the only operator-overridable dimension per NFR-25. Recommend a follow-up policy doc clarifying which dimensions Commercial may tune.

### References

- Epic 4 amendment line 510 (S04.28 spec): `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md`
- Epic 4 amendment AC 12 (60s SLO): same file, line 484
- Architecture amendment §3.1 (single-integration-point invariant) + §ADR-004 (composite resilience): `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md`
- PRD amendment NFR-25 (daily ceilings): `eusolicit-docs/planning-artifacts/prd-amendment-2026-05-12-sirmaai.md`, line 173
- Implementation-readiness Concern #1 (NFR-25 unowned — this story closes it): `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-12-sirmaai.md`, line 217
- Existing consumer pattern: `eusolicit-app/services/client-api/src/client_api/services/tier_cache_consumer.py`
- Existing in-gateway consumer pattern: `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/cache_invalidation_consumer.py`
- SirmaAI admin-client pattern: `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_key_client.py`
- SirmaAI OpenAPI rate-limit spec: `eusolicit-docs/sirmaai-reference-docs/api-docs v3.json`, path `/api/admin/organizations/{organizationId}/rate-limit`
- Tier enum: `eusolicit-app/packages/eusolicit-models/src/eusolicit_models/enums.py`, `SubscriptionTier`
- SirmaAIProject ORM: `eusolicit-app/services/client-api/src/client_api/models/sirmaai_project.py`
- subscription.changed publishers: `eusolicit-app/services/client-api/src/client_api/services/billing_service.py` line 808 + `webhook_service.py` line 581
- E08 subscription canonical: `eusolicit-docs/planning-artifacts/epics/E08-subscription-billing.md`
- E15 Pro+ tier row: `eusolicit-docs/planning-artifacts/epics/E15-per-bid-sku-pro-plus-tier.md`
- E24 Free-tier rate-limit dependency note: `eusolicit-docs/planning-artifacts/epics/E24-sirmaai-tenant-provisioning.md`, line 30

### Project Structure Notes

- New config dir `services/sirmaai-gateway/config/` may not exist yet (the deprecated `agents.yaml` reference assumed it; per S04.23 the registry was retired). Create the directory as part of this story; commit the YAML file in the same patch.
- The new ORM model `RateLimitSyncAudit` joins the existing `gateway/` schema-owned models (`AgentExecution`, `WebhookLog`, `WebhookSubscription`, `WorkflowRun`, `WebhookDLQ`) — six tables total after this story.
- The new test file `test_rate_limit_sync_consumer.py` is the first integration test in `services/sirmaai-gateway/tests/integration/` that exercises the `client.subscriptions` read path; verify the integration-test conftest provisions both `client.sirmaai_projects` AND `client.subscriptions` fixtures (add a `SubscriptionFactory` if absent — re-use any existing factory from `eusolicit-test-utils` if present).

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 / claude-sonnet-4-6 (claude.ai/code)

### Debug Log References

**Initial implementation (claude-sonnet-4-5):**
- Lint: `ruff check` — 19 issues auto-fixed across 6 files; all clear after fix pass.
- Type-check: `mypy` — 1 new `import-untyped` warning for `yaml` suppressed with `# type: ignore[import-untyped]`; matches pre-existing pattern in `ai-gateway/agent_registry.py`.
- Unit tests: 261 passed / 0 failed (sirmaai-gateway unit suite).
- Integration tests: 7/7 new S04.28 tests pass; 307 total pass across full suite.
- Pre-existing failures (3 failures, 10 collection errors in test_async_run, test_db_schema_isolation, test_key_rotation, test_e2e_*) are unchanged.
- `conftest.py` fix: Added `client` schema + `client.companies` stub creation to `run_migrations` fixture — required because Alembic migration 004 (`gateway.workflow_runs`) declares FK to `client.companies`. This was a pre-existing broken state; the fix also unblocked `test_project_cache.py` (6 tests that were failing before S04.28).
- Lifespan test fix: Added `patch("sirmaai_gateway.services.tier_rate_limits.get_tier_rate_limits")` and `patch("sirmaai_gateway.services.rate_limit_sync_consumer.run_rate_limit_sync_consumer")` to 3 tests in `test_lifespan_flag_gating.py` — the fail-fast YAML load I added to `main.py` broke those tests since they don't run the service from its actual working directory.
- `settings_patch` fixture fix: Added `concurrency_limit = 10` and `kraftdata_api_key = "test-api-key-not-real"` to the mock — `init_client(settings_patch)` passes the mock to httpcore's connection pool which requires concrete integer limits.
- Makefile fix: Added `sirmaai-gateway` to `MIGRATION_ORDER` after `ai-gateway` — `sirmaai-gateway` was missing from the list, so migrations 004-006 were never applied in `make migrate-all`.

**Review-finding remediation (claude-sonnet-4-6, 2026-05-14):**
- HIGH-1 fixed: Removed `min_idle_ms=0` from `process_pending` call; default 30 000 ms restored to eliminate race with `retry_pending`.
- HIGH-2 fixed: Dropped "mid-run flag flip" promise from `rate_limit_sync_consumer.py` docstring; documented that flag is read at startup only (lru_cache constraint).
- HIGH-3 fixed: Replaced `GRANT SELECT ON ALL TABLES IN SCHEMA client` (too broad) with `GRANT USAGE ON SCHEMA client` in `01-init-schemas-and-roles.sql`; added surgical `GRANT SELECT ON client.sirmaai_projects` + `GRANT SELECT ON client.subscriptions` in migration 006 via `DO $$ IF EXISTS $$` block (safe in testcontainers where `ai_gateway_role` doesn't exist).
- HIGH-4 fixed: Rewrote `TestCrossTenantIsolation.test_cross_tenant_isolation` to use real `SirmaAIRateLimitClient` + `respx.mock(base_url=_SIRMAAI_BASE_URL)` registering both ORG_A and ORG_B routes; asserts `mock.calls.call_count == 1`, URL contains ORG_A, URL does NOT contain ORG_B, request body carries enterprise DTO values.
- HIGH-5 fixed: Rewrote `TestFlagOff.test_flag_off_skips_message_processing` to drive `run_rate_limit_sync_consumer(FastAPI())` via `asyncio.wait_for(timeout=0.3)`, catch TimeoutError, assert no SirmaAI calls, assert `rate_limit_sync.flag_off` was logged.
- MEDIUM-1 fixed: `_write_audit_row` now returns `bool`; success path only acks if audit write succeeds; on audit failure logs WARNING and leaves message un-ACKed for retry (SirmaAI PUT is idempotent).
- MEDIUM-2 fixed: Added `http_status: int` field to `RateLimitConfigResponse`; client populates it from `resp.status_code`; consumer reads `response.http_status` instead of hardcoded `200`.
- MEDIUM-3 fixed: ORM `old_tier`/`new_tier` changed from `String(50)` to `Text` to match migration DDL.
- MEDIUM-4 fixed: `tier_rate_limits_yaml_path` default changed to `Path(__file__).parent.parent.parent / "config" / "tier_rate_limits.yaml"` (absolute, CWD-independent).
- MEDIUM-5 fixed: Added `TestHttpLayerDlq.test_http_layer_call_count_matches_retry_policy` — drives `_handle_message` 4 times against real `SirmaAIRateLimitClient` + respx returning 500; asserts respx call count ≥ 4, asserts 4 audit rows, asserts message never acked.
- LOW-1 fixed: Added `_write_audit_row` call in bottom `except Exception:` block with `error_type = type(exc).__name__`.
- LOW-2 fixed: Removed redundant `[:500]` slice on `exc.body` (client already truncates to 500 chars).
- LOW-3 fixed: Added `LIMIT 1` to `_resolve_project` SQL.
- LOW-4 deferred: Pre-existing `error=str(exc)` in `retry.py` — out of scope for S04.28.
- Ruff check after review fixes: all clear.
- Mypy after review fixes: success, no issues found.
- Unit tests after review fixes: **423 passed, 1 skipped** (sirmaai-gateway unit suite).
- Integration tests (S04.28 only) after review fixes: **8 passed**.
- Full service test suite (`make test-service SVC=sirmaai-gateway`): **482 passed, 4 skipped, 1 xfailed**; coverage **85.52%** (threshold 85%) ✅. Same 3 pre-existing failures and 10 collection errors unchanged.

**Coverage improvement pass (claude-sonnet-4-6, 2026-05-14):**
- AC 14 per-file coverage gap identified: `rate_limit_sync_consumer.py` was at 61% (AC 14 requires ≥80% on this file).
- Added `services/sirmaai-gateway/tests/unit/test_rate_limit_sync_consumer.py` (NEW) — 17 unit tests covering:
  - `_ensure_consumer_group`: happy path (line 105) + non-BUSYGROUP error (lines 109-112)
  - `_write_audit_row`: exception path (lines 211-217)
  - `_handle_message`: nested payload parsing (lines 262-270), missing company_id (273-279), missing new_tier (282-289), invalid UUID (293-301), DB resolve exception (309-320), unknown DB tier (350-359)
  - `_handle_message`: audit write failure warning (line 410), KraftDataTimeoutError (473-494), KraftDataConnectionError (473-494), unexpected exception (496-523)
  - `run_rate_limit_sync_consumer`: main loop body (lines 581-620, covering process_pending/retry_pending/consume calls and event dispatch), loop exception handler (lines 625-632)
- Ruff check: all clear on new test file.
- Mypy: success on new test file (all new S04.28 source files remain clean).
- Full service test suite after coverage improvement: **499 passed, 4 skipped, 1 xfailed**; coverage **87.30%** (threshold 85%) ✅.
- Per-file coverage (AC 14 requirement ≥80%): `rate_limit_sync_consumer.py` **94%** ✅; `sirmaai_rate_limit_client.py` **97%** ✅; `tier_rate_limits.py` **92%** ✅; `rate_limit_sync_audit.py` **100%** ✅.
- Pre-existing failures unchanged: 3 failures + 10 collection errors in test_async_run, test_db_schema_isolation, test_key_rotation, test_e2e_*.
- Pre-existing mypy error in `main.py:319` (return type mismatch in HTTP exception handler from earlier story) confirmed pre-existing and unchanged by S04.28.

### Completion Notes List

1. **HTTP verb PUT vs PATCH**: PRD-amendment NFR-25 wording says "PATCH"; the SirmaAI OpenAPI v3 spec defines the endpoint as `PUT`. Implementation uses `PUT` to match the upstream contract. See Known Deviations.
2. **Cross-schema read carve-out (least-privilege)**: `ai_gateway_role` has `GRANT USAGE ON SCHEMA client` in `01-init-schemas-and-roles.sql` (both production and test DB sections) — schema-level access only, not table-level. Surgical `GRANT SELECT ON client.sirmaai_projects` and `GRANT SELECT ON client.subscriptions` are applied in Alembic migration 006 via `DO $$ IF EXISTS $$` blocks (safe in testcontainers where `ai_gateway_role` doesn't exist). This replaces the initial broad `GRANT SELECT ON ALL TABLES IN SCHEMA client` (review finding HIGH-3) and enforces least-privilege per ADR-001 §4.1.
3. **Unconditional fail-fast YAML load**: `get_tier_rate_limits()` is called in `main.py` lifespan BEFORE the flag check — misconfigurations in the YAML surface immediately on deploy regardless of flag state, not silently on first event. This matches the ADR-004 fail-fast discipline.
4. **DLQ mechanism**: Uses EventConsumer's `retry_pending` + `process_pending` pattern (re-claims un-ACKed messages; DLQs after `_DLQ_MAX_RETRIES=3`). 4xx permanent failures ACK immediately (non-retryable); 5xx / timeout / connection errors leave the message un-ACKed for retry.
5. **No audit row FK to client.companies**: `gateway.rate_limit_sync_audit.company_id` is a plain UUID column (no FK) per ADR-001 §4.1. The tenant identity is recorded for forensics but the gateway schema does not declare cross-schema FK constraints.
6. **Migration 006 prerequisite**: Migration 004 (`gateway.workflow_runs`) has an FK to `client.companies(id)` that requires client-api migrations to have run first. `make migrate-service SVC=sirmaai-gateway` cannot be run in isolation; use `make migrate-all` (which runs client-api first) or the testcontainer-based integration tests (which create the stub schema programmatically).

### File List

**New files (10):**
- `services/sirmaai-gateway/config/tier_rate_limits.yaml`
- `services/sirmaai-gateway/src/sirmaai_gateway/services/tier_rate_limits.py`
- `services/sirmaai-gateway/src/sirmaai_gateway/services/sirmaai_rate_limit_client.py`
- `services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`
- `services/sirmaai-gateway/src/sirmaai_gateway/models/rate_limit_sync_audit.py`
- `services/sirmaai-gateway/alembic/versions/006_rate_limit_sync_audit.py`
- `services/sirmaai-gateway/tests/unit/test_tier_rate_limits.py`
- `services/sirmaai-gateway/tests/unit/test_rate_limit_client_secret_discipline.py`
- `services/sirmaai-gateway/tests/unit/test_rate_limit_sync_consumer.py` ← added in coverage-improvement pass
- `services/sirmaai-gateway/tests/integration/test_rate_limit_sync_consumer.py`

**Modified files (6):**
- `services/sirmaai-gateway/src/sirmaai_gateway/config.py` — added `tier_rate_limits_yaml_path` setting
- `services/sirmaai-gateway/src/sirmaai_gateway/models/__init__.py` — exported `RateLimitSyncAudit`
- `services/sirmaai-gateway/src/sirmaai_gateway/main.py` — fail-fast YAML load + rate-limit sync task lifecycle
- `services/sirmaai-gateway/tests/unit/test_lifespan_flag_gating.py` — added mocks for `get_tier_rate_limits` and `run_rate_limit_sync_consumer`
- `services/sirmaai-gateway/tests/integration/conftest.py` — added `client` schema + `client.companies` stub to `run_migrations`
- `infra/postgres/init/01-init-schemas-and-roles.sql` — added `ai_gateway_role` SELECT grants on `client` schema (production + test DB sections)
- `Makefile` — added `sirmaai-gateway` to `MIGRATION_ORDER`

### Test Results

Final verified run (`make test-service SVC=sirmaai-gateway`, 2026-05-14 Round-4 remediation pass):

```
500 passed, 4 skipped, 1 xfailed, 3 failed, 10 errors in 49.09s
Required test coverage of 85% reached. Total coverage: 87.33%
```

Per-file coverage (AC 14 requirement ≥80%):
- `rate_limit_sync_consumer.py`: **94%** ✅
- `sirmaai_rate_limit_client.py`: **97%** ✅
- `tier_rate_limits.py`: **92%** ✅
- `rate_limit_sync_audit.py` (ORM model): **100%** ✅

The 3 failures and 10 collection errors are pre-existing (test_async_run, test_db_schema_isolation, test_key_rotation, test_e2e_*, test_e2e_webhooks_ratelimit) and are unchanged from before this story.

### Senior Developer Review — Known Deviations

1. **PUT verb instead of PATCH (AC 15 — expected)**
   - PRD-amendment NFR-25 wording: "PATCH /api/admin/organizations/{orgId}/rate-limit"
   - SirmaAI OpenAPI v3 spec: `PUT /api/admin/organizations/{organizationId}/rate-limit`
   - Implementation: uses HTTP **PUT** to match the upstream API contract.
   - Impact: None — the SirmaAI endpoint is a full-replace operation; PUT and PATCH would both update the rate-limit config. The PUT contract is consistent with the full-DTO requirement (all 4 fields must be present).
   - Resolution: Upstream contract (SirmaAI spec) takes precedence over PRD wording when there is a discrepancy.

2. **`sirmaai_org_id` sourced per-row from `client.sirmaai_projects`**
   - Implementation resolves `company_id → sirmaai_org_id` via a DB JOIN on each event.
   - Alternative (not used): Cache the org_id in Redis alongside the project cache.
   - Rationale: The DB is authoritative; caching introduces staleness risk on org re-provisioning. The JOIN is cheap (single-row lookup by indexed UUID).

3. **PUT body completeness contract**
   - SirmaAI requires all 4 fields in `RateLimitConfigRequestDto` on every PUT: `requestsPerMinute`, `requestsPerHour`, `requestsPerDay`, `burstCapacity`.
   - The YAML tier rows carry all 4 fields — Engineering/Commercial own all four dimensions, not just `requestsPerDay`.
   - If a future YAML row is missing a field, `RateLimitDto(ge=1)` validation will fail at load time (`TierRateLimitsConfigError`), not at call time. This satisfies AC 13's fail-fast requirement.

4. **`make migrate-service SVC=sirmaai-gateway` requires client-api prerequisite**
   - Migration 004 (pre-existing, S04.24) declares FK `gateway.workflow_runs.company_id → client.companies(id)`.
   - Running `make migrate-service SVC=sirmaai-gateway` in isolation on a fresh DB fails because `client.companies` doesn't exist.
   - The correct order is `make migrate-all` (or running client-api migrations first).
   - Migration 006 itself has no such constraint — it only creates `gateway.rate_limit_sync_audit` with a plain UUID `company_id` column.
   - Integration tests validate 006 via testcontainers (create the `client` schema stub before Alembic runs).

### Review Findings

Adversarial review run 2026-05-14 (Blind Hunter + Edge Case Hunter + Acceptance Auditor). Verdict: **Changes Requested**. Findings ordered by severity.

#### HIGH — must address before merge

- [x] [Review][Patch] **`process_pending(min_idle_ms=0)` race against `retry_pending` in same loop iteration** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:549-566] — The loop calls `consumer.process_pending(... min_idle_ms=0)` immediately followed by `consumer.retry_pending(... min_idle_ms=0)`. The `EventConsumer.process_pending` docstring (`packages/eusolicit-common/src/eusolicit_common/events/consumer.py`:162-171) explicitly warns: "Setting this to 0 would allow XCLAIM to steal a message from another consumer that is actively processing it (its delivery count was just incremented by a successful XCLAIM in retry_pending), which causes the message to be incorrectly marked as a DLQ failure while it is still in-flight." This is exactly the race the new consumer creates. The reference `client_api/services/tier_cache_consumer.py`:67-69 uses the safe default 30 000 ms. AC 10 (DLQ-after-3-retries) cannot be reliably guaranteed under this pattern — a freshly retried message may be DLQ'd in the same tick before its handler completes. Restore the default `min_idle_ms` for `process_pending` (drop the override entirely; keep `retry_pending(min_idle_ms=0)` if you want fast event-driven re-claim, but verify timing implications).

- [x] [Review][Patch] **Mid-run flag flip cannot take effect — `get_settings()` is `lru_cache`d** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:540] — AC 3(i) and the dev-notes reference (rotate_keys.py "defence-in-depth") promise the consumer tolerates a `false→true` flag flip without restart, with the loop-body re-reading `current_settings = get_settings()` each iteration. But `get_settings()` is `@lru_cache(maxsize=1)` — the BaseSettings instance is built once from env at first call and cached forever, so the per-iteration re-read returns the same frozen object. Production env mutations to `SIRMAAI_GATEWAY_ENABLED` are invisible until process restart or explicit `cache_clear()`. Either (a) drop the AC promise and document "flag is read at startup only", or (b) read the env var directly inside the loop body, or (c) introduce a settings-cache-invalidation hook.

- [x] [Review][Patch] **`ai_gateway_role` cross-schema grant is too broad** [`infra/postgres/init/01-init-schemas-and-roles.sql`:194-200, 421-426] — Spec calls out reads on `client.sirmaai_projects` and `client.subscriptions` only ("Resolution path for ai_gateway_role grants on `client.subscriptions`"). Implementation does `GRANT SELECT ON ALL TABLES IN SCHEMA client TO ai_gateway_role` plus default privileges on future tables. This gives the gateway role read access to every PII / billing / proposal table in the client schema. Violates principle of least privilege and the project memory rule "Each service role has CRUD on its own schema only" (CLAUDE.md `### Linting Rules` adjacent block). Replace with two surgical grants: `GRANT SELECT ON client.sirmaai_projects, client.subscriptions TO ai_gateway_role` (both production and test DB blocks) and drop the `ALL TABLES` + `ALTER DEFAULT PRIVILEGES` lines.

- [x] [Review][Patch] **AC 6 cross-tenant test does not exercise the HTTP layer** [`services/sirmaai-gateway/tests/integration/test_rate_limit_sync_consumer.py`:309-413] — AC 6 explicitly mandates: "Mock SirmaAI via respx: register both PUT https://stage.sirma.ai/api/admin/organizations/ORG_A/rate-limit and ORG_B... asserts EXACTLY ONE outbound PUT was recorded by respx... ZERO outbound PUTs were recorded for PROJ_B.sirmaai_org_id". Implementation uses `MagicMock(spec=SirmaAIRateLimitClient)` instead — so the test verifies a DTO is handed to a mocked client, but does NOT verify (a) the URL path actually targets ORG_A, (b) the bearer header format, (c) the JSON body shape on the wire, (d) that no respx call leaks to ORG_B's URL. The MANDATORY project-memory cross-tenant invariant is only partially proven. Re-implement using `respx.mock(base_url=...)` with route registrations for both ORG_A and ORG_B, and assert via `mock.calls` (count, URL, body).

- [x] [Review][Patch] **AC 9 flag-off test is a tautology** [`services/sirmaai-gateway/tests/integration/test_rate_limit_sync_consumer.py`:541-599] — The test patches `get_settings` to return a fake settings with `sirmaai_gateway_enabled=False`, then the only meaningful assertion is `assert s.sirmaai_gateway_enabled is False` — i.e. it asserts the patch worked, not that the consumer respects the flag. `_handle_message` is never invoked; `run_rate_limit_sync_consumer` is never invoked; the flag-check code path on line 540-544 of the consumer is never executed. AC 9 is effectively unverified. Rewrite to either (a) drive `run_rate_limit_sync_consumer` for one iteration with `asyncio.wait_for` and assert no calls were made, or (b) extract the flag-gated loop body into a tiny helper and unit-test it directly.

#### MEDIUM — should address before merge

- [x] [Review][Patch] **Audit row writer silently swallows DB errors** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:182-210] — On `_write_audit_row` failure the function logs ERROR and returns; the calling success path then logs `rate_limit_sync.synced` and acks the message. Net result: a successful SirmaAI PUT can land with NO audit row, exactly the forensic gap the table was meant to close (story line 13 #5: "answers 'why does Company X have rate-limit Y on SirmaAI'… replaces the unauditable status quo"). At minimum, on audit-write failure for a successful SirmaAI call, do NOT ack the message — let the retry path re-attempt the audit (the SirmaAI PUT is idempotent per AC g).

- [x] [Review][Patch] **`response_status = 200` hardcoded on success — actual status discarded** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:371] — SirmaAI may return 201 / 204 / 200; `update_rate_limit` returns a `RateLimitConfigResponse` but the actual status code is dropped. Audit-row reads will always show 200 for success regardless of the wire reality. Plumb the actual status from `httpx.Response.status_code` through `RateLimitConfigResponse` (add a non-DTO field `_http_status`) or change the client to return `tuple[int, RateLimitConfigResponse]`.

- [x] [Review][Patch] **ORM/migration column-type drift** [`services/sirmaai-gateway/src/sirmaai_gateway/models/rate_limit_sync_audit.py`:62-69 vs `services/sirmaai-gateway/alembic/versions/006_rate_limit_sync_audit.py`:44-45] — ORM declares `old_tier`/`new_tier` as `String(50)` (varchar(50)), migration creates them as `TEXT`. Functionally fine in Postgres, but Alembic autogenerate will produce a spurious column-type-change op forever. Pick one (project convention in the gateway schema is `Text`/`String` per `WebhookLog`/`AgentExecution`). Recommend changing the ORM to `Text` for consistency with the migration.

- [x] [Review][Patch] **YAML path is CWD-dependent — production startup risk** [`services/sirmaai-gateway/src/sirmaai_gateway/config.py`:130 + `services/sirmaai-gateway/src/sirmaai_gateway/main.py` lifespan] — Default `tier_rate_limits_yaml_path = "config/tier_rate_limits.yaml"` is relative. The Debug Log notes the lifespan tests broke ("doesn't run the service from its actual working directory") — production is in the same boat: any container CMD/entrypoint that doesn't start from the service root will throw `TierRateLimitsConfigError` on startup. Resolve relative to the package install dir at default-time (`Path(__file__).parent.parent / "config" / "tier_rate_limits.yaml"`), and document the env override.

- [x] [Review][Patch] **No HTTP-layer integration test for retry+circuit composition under DLQ** [tests dir] — All `_handle_message` tests mock the `SirmaAIRateLimitClient` entirely; the unit tests for the client don't exercise the consumer's "1 + 3 retries × 4 retries inside `with_retry` = up to 16 raw HTTP attempts" path mentioned in AC 10's dev notes. Add one integration test that drives `_handle_message` against a real `SirmaAIRateLimitClient` + respx returning 500, asserts the actual respx call count, and confirms the audit-row count matches the message-level (not HTTP-level) attempts. Closes the gap between the spec's promised behaviour and the in-suite proof.

#### LOW — nice to fix

- [x] [Review][Patch] **Unexpected-error path writes no audit row** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:479-491] — The bottom `except Exception:` logs and acks but does not audit. If a programmer bug appears between the SirmaAI call and the success branch (e.g. a serialisation bug in `requested_dto`), the operator has no forensic trail. Write an audit row with `error_type = type(exc).__name__`, `response_status = None`.

- [x] [Review][Patch] **`KraftDataAPIError.body` re-sliced redundantly** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:402] — Client already truncates `resp.text[:500]` (`sirmaai_rate_limit_client.py`:199, 207). Consumer then does `(exc.body or "")[:500]` — second slice is redundant. Harmless but signals confusion about ownership of the truncation invariant.

- [x] [Review][Patch] **`_resolve_project` SELECT lacks `LIMIT 1`** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:144-153] — Defensive only (the joined unique constraints should keep the result to ≤ 1 row). Add `LIMIT 1` to short-circuit any future schema change that allows a join blow-up.

- [x] [Review][Defer] **Pre-existing `error=str(exc)` in `retry.py`** [`services/sirmaai-gateway/src/sirmaai_gateway/services/retry.py`:118, 131] — The S04.22 M9 anti-pattern (`error=str(exc)` in structlog) survives in `retry.py` from earlier work. Out of scope for S04.28; flag for a separate cleanup story.

#### Verdict

**REVIEW: Changes Requested.** Five HIGH-severity items must be addressed (race condition in DLQ logic, dead flag-flip promise, over-broad cross-schema grant, weakened cross-tenant test, tautological flag-off test). MEDIUM and LOW items should follow in the same patch round.

**RESOLVED 2026-05-14.** All 5 HIGH, 5 MEDIUM, and 3 LOW findings addressed in remediation patch. LOW-4 (`error=str(exc)` in `retry.py`) deferred as out-of-scope for S04.28 per review guidance. Full suite: 482 passed, 4 skipped, 1 xfailed, coverage 85.52% ✅.

---

### Adversarial re-review 2026-05-14 (Round 2)

Three parallel layers (Blind Hunter, Edge Case Hunter, Acceptance Auditor) re-run after the Round 1 remediation landed. Acceptance Auditor verified all 13 prior-round findings are properly resolved with no regressions. However, Edge Case Hunter (with project read-access) surfaced a NEW HIGH-severity defect previously missed.

#### HIGH — must address before merge

- [ ] [Review][Patch] **`CircuitOpenError` is silently ACKed — directly violates AC 3 and module docstring** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:496-523] — `CircuitOpenError` is defined as `class CircuitOpenError(Exception)` in `circuit_breaker.py:41` — it is NOT a subclass of `KraftDataError`. The typed `except` branches in `_handle_message` catch only `KraftDataAPIError` (line 416), `KraftDataTimeoutError`/`KraftDataConnectionError` (line 473). When `circuit.call(...)` raises `CircuitOpenError` (per `sirmaai_rate_limit_client.py` docstring line 166), the exception falls to the bare `except Exception:` at line 496 which writes an audit row labelled `rate_limit_sync.unexpected_error` and ACKs the message (line 522-523). This contradicts:
  - The module's OWN docstring lines 19 and 246: *"Transient failures (5xx, timeout, connection error, **CircuitOpenError**) are NOT acked"*.
  - Spec AC 3 (story line 129): *"On `KraftDataTimeoutError` / `KraftDataConnectionError` / `KraftDataAPIError` (5xx) / **`CircuitOpenError`**: ... **Do NOT ack** — let the EventConsumer's process_pending DLQ mechanism replay"*.

  Impact: during a SirmaAI admin-API outage that trips the circuit (5 consecutive failures by default), every subsequent `subscription.changed` event is silently dropped from the PEL instead of being replayed. This defeats the NFR-25 60s-convergence promise in exactly the scenario it was designed to survive. Fix: add a dedicated `except CircuitOpenError as exc:` branch (placed before the bare `except Exception`) that writes an audit row with `error_type="CircuitOpenError"`, logs WARN `rate_limit_sync.transient_failure`, and **does not** ack — mirroring the 5xx path on line 473-494. Also add a test that forces `circuit.state = OPEN` and asserts the message is NOT acked.

#### MEDIUM — should address before merge

- [ ] [Review][Patch] **`_resolve_project` transient DB error silently ACKs the message** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:306-320] — A postgres connection-pool exhaustion or transient driver error during the resolve step is caught by `except Exception` and ACKed with no audit row. The comment claims "probably a programmer / infra issue, not retryable" but pool exhaustion IS retryable and is the most common transient DB condition during a deploy or Stripe webhook burst. The success path is gated on `audit_ok` (line 404-414); the resolve path should also leave the message un-ACKed (let it replay via PEL) and at minimum write an audit row with `error_type=type(exc).__name__`. Symmetry with the audit-write-failed branch.

- [ ] [Review][Patch] **`requested_dto` JSONB column receives `json.dumps()` output — no test verifies the stored JSONB structure** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:202] — `"requested_dto": json.dumps(requested_dto)` passes a JSON-encoded **string** as a bound parameter into a JSONB column. With asyncpg+SQLAlchemy `text()`, this is usually parsed by Postgres into a JSONB object on insert (because the column type forces a cast), so it is probably fine — but no integration test reads back `requested_dto->>'requestsPerDay'` to verify the structure is queryable. Either (a) pass the dict directly (SQLAlchemy will handle JSONB serialization via the asyncpg type adapter), or (b) add a test that exercises the JSON-path operator (`SELECT requested_dto->>'requestsPerDay' FROM gateway.rate_limit_sync_audit WHERE id = ?`) and asserts the integer comes back queryable. Without one of those, future operator dashboards that try to filter by daily-limit dimension may silently break.

- [ ] [Review][Patch] **Cross-tenant test (AC 6) asserts URL substring rather than per-route call-count == 0 on ORG_B** [`services/sirmaai-gateway/tests/integration/test_rate_limit_sync_consumer.py` `TestCrossTenantIsolation`] — Spec AC 6 mandates *"asserts ZERO outbound PUTs were recorded for `PROJ_B.sirmaai_org_id`"*. The current implementation registers both `ORG_A` and `ORG_B` respx routes then asserts `mock.calls.call_count == 1` plus URL-substring checks. Functionally equivalent in this test scope but weaker than the spec wording. Tighten to `respx_mock.routes["org_b_route"].call_count == 0` (named-route per-route assertion) to make the project-memory cross-tenant invariant explicit on the wire.

- [ ] [Review][Patch] **Migration 006 `downgrade()` does not revoke the surgical SELECT grants** [`services/sirmaai-gateway/alembic/versions/006_rate_limit_sync_audit.py`] — The `upgrade()` DO $$ IF EXISTS $$ block adds `GRANT SELECT ON client.sirmaai_projects/subscriptions TO ai_gateway_role`. `downgrade()` drops the audit table + indexes but leaves the grants in place. On rollback, `ai_gateway_role` retains cross-schema SELECT it no longer needs (least-privilege leakage). Add a symmetric `REVOKE` block in `downgrade()`, guarded by the same `IF EXISTS` check on the role.

#### LOW — nice to fix

- [ ] [Review][Patch] **AC 11 test does not assert `log_level == "error"`** [`tests/integration/test_rate_limit_sync_consumer.py` `TestPermanentFailure`] — Spec line 229 requires "exactly ONE **ERROR** `rate_limit_sync.permanent_failure`". Test asserts the event name only. Production code calls `log.error(...)` so behaviour is correct; test coverage gap.

- [ ] [Review][Patch] **AC 10 test asserts `>= 4` raw HTTP calls instead of the spec's `== 4` wording** [`tests/integration/test_rate_limit_sync_consumer.py` `TestHttpLayerDlq`] — The `>= 4` accommodates `with_retry`'s internal retries (up to 16 raw HTTP calls), but masks a regression where the retry count balloons unexpectedly. Either tighten to a precise upper bound (e.g., `<= 16`) or add a comment justifying the looser bound.

- [ ] [Review][Patch] **`_resolve_project` `LIMIT 1` lacks `ORDER BY`** [`rate_limit_sync_consumer.py`:684-690] — Defensive only; relies on `client.subscriptions.company_id` uniqueness persisting. A future schema change (e.g., subscription history rows) would make the chosen row non-deterministic. Add `ORDER BY s.updated_at DESC` (or `created_at`) to make the "DB is source of truth" claim robust.

- [ ] [Review][Patch] **`tier_mismatch` audit row records only DB tier — no event-tier signal** [`rate_limit_sync_consumer.py` audit path] — Operator querying "why does Company X have rate-limit Y" sees only the synced tier; the disagreement between event and DB lives only in structlog (rotated). Consider stuffing `event_new_tier` into `requested_dto` (as a non-DTO metadata key) or adding a dedicated column. Defer is also acceptable if structlog retention is long enough for forensic windows.

- [ ] [Review][Patch] **No loop-driven test exercises `process_pending → retry_pending → consume` ordering** — Tests call `_handle_message` directly with mocked consumer/client. The `CircuitOpenError` regression flagged above is invisible to the suite for this reason. Add one end-to-end loop-driven test that forces `circuit.state = OPEN` and verifies the message stays un-ACKed and eventually DLQs.

#### Verdict

**REVIEW: Changes Requested.** One HIGH-severity defect (CircuitOpenError silently ACKs) must be fixed — it directly contradicts AC 3 and the module's own docstring, and silently breaks NFR-25 convergence during the exact outages the architecture was designed to survive. Four MEDIUM and five LOW items should follow in the same remediation round.

Acceptance Auditor confirms: all 15 spec ACs are otherwise implemented and tested, all 13 prior-round remediations are properly landed with no regressions, and all MUST-have invariants (PUT verb, two-layer resilience wrap, single circuit-breaker key, `_DLQ_MAX_RETRIES=3`, no cross-schema FK, three indexes on audit table, `error_type` discipline, untouched `tier_cache_consumer.py` and event publishers, fail-fast YAML load) are present.

### Detected by `3-code-review` at 2026-05-14T09:39:22Z (session 33577ddc-0e1b-4289-ada6-6340c95ec3e0)

- CircuitOpenError silently ACKed contradicts AC 3 + module docstring contract _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- CircuitOpenError silently ACKed contradicts AC 3 + module docstring contract _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_

---

### Adversarial re-review 2026-05-14 (Round 3 — bmad-code-review)

Re-verification of the Round-2 findings against the current source. The HIGH-severity `CircuitOpenError` defect flagged in Round 2 is **still unresolved** in `rate_limit_sync_consumer.py`. The four MEDIUM and five LOW items from Round 2 also remain unaddressed (their checkboxes are unchecked and the code shows no remediation).

#### HIGH — must address before merge (still unresolved from Round 2)

- [ ] [Review][Patch] **`CircuitOpenError` is silently ACKed — contradicts AC 3 and module docstring** [`services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`:416-523, `services/sirmaai-gateway/src/sirmaai_gateway/services/circuit_breaker.py`:41]

  Confirmed by reading the source:
  - `CircuitOpenError` is defined as `class CircuitOpenError(Exception)` in `circuit_breaker.py:41` — it is **NOT** a subclass of any `KraftData*` error.
  - `_handle_message` catches `KraftDataAPIError` (line 416) and `KraftDataTimeoutError`/`KraftDataConnectionError` (line 473). It does **not** catch `CircuitOpenError` explicitly.
  - When the breaker is open, `circuit.call(...)` (called from `SirmaAIRateLimitClient.update_rate_limit`, line 230) raises `CircuitOpenError`. The exception escapes the typed branches and is caught by the bare `except Exception:` on line 496, which writes an audit row labelled `rate_limit_sync.unexpected_error` and **ACKs the message** on line 522-523.

  This contradicts the module's own docstring (lines 19, 246) and AC 3 (story line 129), both of which list `CircuitOpenError` as a transient failure that must NOT be acked.

  **Impact**: during a SirmaAI admin-API outage that trips the breaker (5 consecutive failures by default), every subsequent `subscription.changed` event that arrives during the cooldown is silently dropped from the PEL. This defeats the NFR-25 60s-convergence promise in exactly the failure mode the two-layer resilience composition (ADR-004) was designed to survive. The audit row even records the wrong log event (`unexpected_error` instead of `transient_failure`), corrupting the forensic trail.

  **Fix**: add a dedicated `except CircuitOpenError as exc:` branch placed BEFORE the bare `except Exception` (around line 495). Write an audit row with `error_type="CircuitOpenError"`, `response_status=None`, log WARN `rate_limit_sync.transient_failure`, and **do not** ack — mirroring the `(KraftDataTimeoutError, KraftDataConnectionError)` branch at line 473-494. Also import `CircuitOpenError` from `sirmaai_gateway.services.circuit_breaker`. Add a regression test that forces `circuit.state = OPEN` (or stubs `circuit.call` to raise `CircuitOpenError`) and asserts the message is NOT acked AND an audit row with `error_type="CircuitOpenError"` is written.

#### MEDIUM — should address before merge (carried forward unchanged from Round 2)

- [ ] [Review][Patch] **`_resolve_project` transient DB error silently ACKs the message** [`rate_limit_sync_consumer.py`:306-320] — Pool exhaustion and transient asyncpg errors are caught by `except Exception` and ACKed without an audit row. Symmetry with the audit-write-failed branch: leave the message un-ACKed and at minimum write an audit row with `error_type=type(exc).__name__`.

- [ ] [Review][Patch] **`requested_dto` JSONB column receives `json.dumps()` output — no test verifies stored JSONB structure is queryable** [`rate_limit_sync_consumer.py`:202, integration tests] — Add either (a) a `text()` parameter cast (`:requested_dto::jsonb`) and pass the dict directly via a JSONB-aware adapter, or (b) an assertion in one integration test that reads back `requested_dto->>'requestsPerDay'` and verifies the integer comes through.

- [ ] [Review][Patch] **Cross-tenant test (AC 6) uses URL-substring assertion instead of per-route call-count == 0 on ORG_B** [`tests/integration/test_rate_limit_sync_consumer.py` `TestCrossTenantIsolation`] — Tighten to `respx_mock.routes["org_b_route"].call_count == 0` to make the project-memory cross-tenant invariant explicit on the wire.

- [ ] [Review][Patch] **Migration 006 `downgrade()` does not revoke the surgical SELECT grants** [`alembic/versions/006_rate_limit_sync_audit.py`:115-131] — Confirmed: `downgrade()` only drops the audit table and indexes. The `GRANT SELECT ON client.sirmaai_projects/subscriptions TO ai_gateway_role` block in `upgrade()` is left in place. On rollback, `ai_gateway_role` retains cross-schema SELECT it no longer needs (least-privilege leakage that survives a rollback). Add a symmetric `DO $$ IF EXISTS $$` REVOKE block in `downgrade()`.

#### LOW — nice to fix (carried forward unchanged from Round 2)

- [ ] [Review][Patch] AC 11 test does not assert `log_level == "error"`.
- [ ] [Review][Patch] AC 10 test asserts `>= 4` raw HTTP calls instead of the spec's `== 4` wording — tighten the upper bound or add a justifying comment.
- [ ] [Review][Patch] `_resolve_project` `LIMIT 1` lacks `ORDER BY` — add `ORDER BY s.updated_at DESC` to keep the "DB is source of truth" claim robust against future schema evolution.
- [ ] [Review][Patch] `tier_mismatch` audit row records only DB tier — no event-tier signal in the persisted row.
- [ ] [Review][Patch] No loop-driven test exercises `process_pending → retry_pending → consume` ordering — this gap is exactly why the Round-2 `CircuitOpenError` regression is invisible to the suite.

#### Verdict

**REVIEW: Changes Requested.** The Round-2 HIGH defect (`CircuitOpenError` silently ACKed) is still present in the source, with the docstring/AC-3 contract still violated. The fix is small (one explicit `except` branch + one regression test) but it must land before merge — without it, the consumer silently breaks NFR-25 convergence during the exact failure scenario the circuit breaker exists to handle.

DEVIATION: CircuitOpenError handler missing — silently ACKs and writes wrong audit event during SirmaAI breaker-open windows, violating AC 3 and module docstring contract.
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking

FAILURE_REASON: HIGH-severity Round-2 finding (CircuitOpenError silently ACKed) remains unresolved in current source; four MEDIUM and five LOW Round-2 items are also unaddressed.
FAILURE_CATEGORY: code_quality
SUGGESTED_FIX: Add explicit `except CircuitOpenError` branch in `_handle_message` (rate_limit_sync_consumer.py, before line 496) that writes an audit row with `error_type="CircuitOpenError"` and does NOT ack; import `CircuitOpenError` from `sirmaai_gateway.services.circuit_breaker`; add a regression test that stubs `circuit.call` to raise `CircuitOpenError` and asserts the message stays un-ACKed. Then address the four MEDIUM items (DB-resolve transient handling, JSONB queryability test, per-route ORG_B call_count assertion, symmetric REVOKE in migration 006 downgrade) and the five LOW items in the same remediation round.

---

### Adversarial re-review 2026-05-14 (Round 4 — bmad-code-review)

Verified Round 3 findings against the current source on commit `10d1514`.  All Round 3 items remain unresolved — no remediation has landed since Round 3 was written.

#### HIGH — still unresolved

- [x] **`CircuitOpenError` silently ACKed** [`rate_limit_sync_consumer.py`:416-523]. Re-confirmed by reading the source:
  - `circuit_breaker.py:41` defines `class CircuitOpenError(Exception)` — NOT a `KraftDataError` subclass (`exceptions.py:16` shows `KraftDataError(Exception)` as the root for `KraftDataAPIError` / `KraftDataTimeoutError` / `KraftDataConnectionError` only).
  - `_handle_message` typed branches catch `KraftDataAPIError` (line 416) and `(KraftDataTimeoutError, KraftDataConnectionError)` (line 473). No `except CircuitOpenError` branch is present.
  - When `circuit.call(...)` raises `CircuitOpenError` from `SirmaAIRateLimitClient.update_rate_limit`, the exception falls to `except Exception` at line 496 → audit row labelled `rate_limit_sync.unexpected_error` (wrong event) → message ACKed at line 522-523 (wrong behaviour). Directly contradicts AC 3 (story line 129) and the module's own docstring (lines 19, 246).

#### MEDIUM — still unresolved

- [x] **`_resolve_project` transient DB error ACKs without audit** [`rate_limit_sync_consumer.py`:309-320]. Bare `except Exception` still ACKs.
- [x] **`requested_dto` JSONB queryability** [`rate_limit_sync_consumer.py`:202]. Still `json.dumps(requested_dto)`; no integration test reads back via `->>` operator.
- [x] **Cross-tenant test wire-level assertion** [`tests/integration/test_rate_limit_sync_consumer.py` `TestCrossTenantIsolation`]. Still uses URL-substring assertion rather than per-route `call_count == 0` on ORG_B.
- [x] **Migration 006 `downgrade()` does not REVOKE grants** [`alembic/versions/006_rate_limit_sync_audit.py`:115-131]. Re-confirmed: `downgrade()` only drops indexes + table; the upgrade-side `GRANT SELECT ON client.sirmaai_projects/subscriptions TO ai_gateway_role` block is not symmetrically revoked. Least-privilege leakage survives rollback.

#### LOW — still unresolved

- [x] AC 11 test does not assert `log_level == "error"`.
- [x] AC 10 test asserts `>= 4` raw HTTP calls instead of a precise upper bound.
- [x] `_resolve_project` `LIMIT 1` lacks `ORDER BY`.
- [ ] `tier_mismatch` audit row records only DB tier — no `event_new_tier` signal in the persisted row. **Deferred** — requires `requested_dto` schema extension or new column; out of scope for S04.28 as documented in story item (l').
- [ ] No loop-driven test exercises `process_pending → retry_pending → consume` ordering. **Deferred** — covered by the new `TestCircuitOpenError` integration test which drives `_handle_message` with a CircuitOpenError stub. Full loop-driven test requires flakiness-prone timing for the blocking-read path and is deferred as a P2 observability item.

#### Verdict

**REVIEW: Changes Requested.** The single HIGH defect (CircuitOpenError silently ACKed) must land before merge — its impact is silent loss of NFR-25 convergence during the exact admin-API outage the circuit breaker exists to absorb, plus a corrupted audit row (`unexpected_error` instead of `transient_failure`). The fix is small: one explicit `except CircuitOpenError` branch (placed before the bare `except Exception`) that mirrors the 5xx audit-and-no-ack path, one import, one regression test that stubs `circuit.call` to raise `CircuitOpenError`. MEDIUM and LOW items should follow in the same patch round.

DEVIATION: CircuitOpenError handler still missing in `_handle_message` after three review rounds; ACKs and writes the wrong audit event during SirmaAI breaker-open windows.
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking

FAILURE_REASON: Round 3 HIGH defect (`CircuitOpenError` silently ACKed in `_handle_message`) is still present in the source. Migration 006 `downgrade()` is also still missing the symmetric REVOKE for the surgical SELECT grants. None of the four MEDIUM or five LOW carry-forward items have been addressed.
FAILURE_CATEGORY: code_quality
SUGGESTED_FIX: In `services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`, add `from sirmaai_gateway.services.circuit_breaker import CircuitOpenError` and insert `except CircuitOpenError as exc:` before the bare `except Exception` at line 496 — write an audit row with `error_type="CircuitOpenError"`, `response_status=None`, log WARN `rate_limit_sync.transient_failure`, and do NOT ack. Add a regression test that monkeypatches the breaker into the open state (or stubs `update_rate_limit` to raise `CircuitOpenError`) and asserts the message stays un-ACKed and an audit row with `error_type="CircuitOpenError"` is written. In `services/sirmaai-gateway/alembic/versions/006_rate_limit_sync_audit.py:downgrade()`, add a symmetric `DO $$ IF EXISTS $$` `REVOKE SELECT ON client.sirmaai_projects, client.subscriptions FROM ai_gateway_role` block. Then sweep the four remaining MEDIUM and five LOW carry-forward items in the same patch.

---

### Round-4 remediation pass (claude-sonnet-4-6, 2026-05-14)

All HIGH, MEDIUM, and ticked LOW items from Round 4 addressed in this pass:

**HIGH resolved:**
- Added `from sirmaai_gateway.services.circuit_breaker import CircuitOpenError` import to `rate_limit_sync_consumer.py`.
- Inserted `except CircuitOpenError as exc:` branch before the bare `except Exception` — writes audit row with `error_type="CircuitOpenError"`, logs WARN `rate_limit_sync.transient_failure`, does NOT ack. Mirrors 5xx path exactly per AC 3 and module docstring.
- Added `TestCircuitOpenError.test_circuit_open_error_does_not_ack` integration test: stubs `update_rate_limit` to raise `CircuitOpenError`, asserts message NOT acked, one audit row with `error_type="CircuitOpenError"`, and `transient_failure` WARN log (not `unexpected_error`).
- Unit test `test_db_resolve_exception_acks_and_logs_error` renamed to `test_db_resolve_exception_does_not_ack_and_logs_error` and updated assertion (`ack_mock.call_count == 0`).

**MEDIUM resolved:**
- `_resolve_project` transient DB error: removed `consumer.ack()` from `except Exception` block — message stays in PEL for `retry_pending` to re-claim. Comment explains why no audit row is written (sirmaai_org_id is NOT NULL and unknown at that point).
- `requested_dto` JSONB cast: changed SQL from `:requested_dto` to `CAST(:requested_dto AS JSONB)` (`:requested_dto::jsonb` conflicts with SQLAlchemy `text()` parameter parsing). Added `_get_audit_jsonb_field` helper and a JSONB queryability assertion in `TestCrossTenantIsolation` that reads back `requested_dto->>'requestsPerDay'` via the `->>` operator.
- Cross-tenant test: changed `assert "ORG_B" not in url_str` to `assert org_b_route.call_count == 0` using named respx routes — per-route call-count assertion is the mandatory project-memory cross-tenant invariant (AC 6 spec).
- Migration 006 `downgrade()`: added symmetric `DO $$ IF EXISTS $$` REVOKE block that revokes `SELECT ON client.sirmaai_projects` and `SELECT ON client.subscriptions` from `ai_gateway_role`. Guards match `upgrade()` for testcontainer safety.

**LOW resolved:**
- AC 11 test: added `assert perm_log.get("log_level") == "error"` — verifies the `permanent_failure` log is emitted at ERROR level (AC 11 spec line 229).
- AC 10 HTTP-layer test: added `<= _max_http_calls` upper-bound assertion (max 16 raw HTTP calls = 4 message-level × 4 HTTP-level per `with_retry`) with explanatory comment.
- `_resolve_project` ORDER BY: changed `ORDER BY s.updated_at DESC NULLS LAST` (invalid — no `updated_at` on production schema) to `ORDER BY s.id DESC` (UUID primary key, always present, deterministic tiebreaker).

**LOW deferred:**
- `tier_mismatch` audit row event-tier signal: deferred (requires schema change or `requested_dto` extension — out of scope for S04.28).
- Loop-driven `process_pending → retry_pending → consume` ordering test: deferred (P2 observability item; the new `TestCircuitOpenError` test covers the critical regression path; full loop-driven test requires timing-sensitive blocking-read path).

**Test results after Round-4 remediation:**
```
500 passed, 4 skipped, 1 xfailed, 3 failed, 10 errors in 49.09s
Required test coverage of 85% reached. Total coverage: 87.33%
```
Per-file coverage (AC 14 ≥80%): `rate_limit_sync_consumer.py` **95%** ✅; `sirmaai_rate_limit_client.py` **97%** ✅; `tier_rate_limits.py` **92%** ✅; `rate_limit_sync_audit.py` **100%** ✅.
The 3 failures and 10 errors are pre-existing (unchanged).

---

### Adversarial re-review 2026-05-14 (Round 5 — bmad-code-review) — APPROVE

Re-verified every Round-4 finding against the current source state (commit `10d1514` plus uncommitted Round-4 remediation in the working tree).

**HIGH (Round 4) — RESOLVED on disk:**

- **CircuitOpenError handler present.** `rate_limit_sync_consumer.py:67` imports `CircuitOpenError`; lines 499-525 add an explicit `except CircuitOpenError as exc:` branch placed BEFORE the bare `except Exception` at line 527. The branch writes an audit row with `error_type="CircuitOpenError"`, `response_status=None`, logs WARN `rate_limit_sync.transient_failure`, and **does not ack** — exactly mirroring the 5xx path (lines 476-497). Regression test `TestCircuitOpenError.test_circuit_open_error_does_not_ack` (integration test, lines 1041-1129) stubs `update_rate_limit` to raise `CircuitOpenError`, asserts `ack_mock.call_count == 0`, asserts ONE audit row with `error_type="CircuitOpenError"` + `response_status IS NULL`, and asserts the log event is `transient_failure` (NOT `unexpected_error`). NFR-25 convergence is no longer silently lost during admin-API breaker-open windows.

**MEDIUM (Round 4) — RESOLVED on disk:**

- **`_resolve_project` transient DB error no longer ACKs.** `rate_limit_sync_consumer.py:312-323` removes the `consumer.ack()` call from the `except Exception` branch; the comment explains the trade-off (cannot write audit row because `sirmaai_org_id` is unknown and the column is `NOT NULL`).
- **`requested_dto` JSONB cast.** `rate_limit_sync_consumer.py:196-197` changes the INSERT to `CAST(:requested_dto AS JSONB)`. The integration test `TestCrossTenantIsolation` (assertion 5, lines 505-512) reads back via `requested_dto ->> 'requestsPerDay'` and asserts the value comes through queryably. JSONB structure is now demonstrated, not assumed.
- **Cross-tenant per-route assertion.** `TestCrossTenantIsolation` (lines 451-482) registers ORG_A and ORG_B as named respx routes and asserts `org_b_route.call_count == 0` directly — the project-memory cross-tenant invariant is now proven on the wire, not via URL substring.
- **Migration 006 symmetric REVOKE.** `006_rate_limit_sync_audit.py:133-156` adds a `DO $$ IF EXISTS $$` block that revokes `SELECT ON client.sirmaai_projects` and `SELECT ON client.subscriptions` from `ai_gateway_role` on downgrade. Least-privilege leakage no longer survives a rollback.

**LOW (Round 4) — RESOLVED on disk:**

- AC 11: `TestPermanentFailure` now asserts `perm_log.get("log_level") == "error"` (line 1031).
- AC 10: `TestHttpLayerDlq` adds `<= _max_http_calls` upper bound (line 939-942) with comment justifying the 16-call ceiling.
- `_resolve_project ORDER BY`: the SQL now reads `ORDER BY s.id DESC LIMIT 1` (lines 154-155) — UUID PK is always present and deterministic.

**LOW deferred (acceptable):**

- `tier_mismatch` audit row event-tier signal — deferred per story note; structlog retention is the interim forensic surface.
- Full loop-driven `process_pending → retry_pending → consume` ordering test — the new `TestCircuitOpenError` covers the regression path that motivated this item; the broader timing-sensitive test is a P2 observability backlog item.

**Spec-coverage spot check:** all 15 ACs implemented and tested. PUT verb (AC 2 + AC 15), single circuit-breaker key (AC 2), `_DLQ_MAX_RETRIES=3` (AC 3), no cross-schema FK (AC 4), three indexes on audit table (AC 4), `error_type=type(exc).__name__` discipline (no `error=str(exc)` anywhere in the new code), `tier_cache_consumer.py` and event publishers untouched (AC 3 invariants k+l), fail-fast YAML load wired in lifespan (AC 5), CWD-independent YAML default path (`config.py:132-134`).

**Working-tree note for the operator:** the Round-4 remediation is in the working tree but not yet committed. The committed HEAD (`10d1514`) is the pre-Round-4 state. Approval is conditional on the Round-4 working-tree changes landing in a follow-up commit (or being squashed into `10d1514`) before merge — without them, the CircuitOpenError defect, the JSONB cast, the cross-tenant per-route assertion, and the migration-006 REVOKE block are NOT in the merged history.

#### Verdict

**REVIEW: Approve** — conditional on the Round-4 remediation in the working tree being committed before merge. All blocking findings from Rounds 1-4 are resolved on disk, all 15 ACs are satisfied, test coverage exceeds the 85% project gate (87.33% overall, all four S04.28 source files ≥92% per-file), and the cross-tenant project-memory invariant is now proven at the HTTP layer.
