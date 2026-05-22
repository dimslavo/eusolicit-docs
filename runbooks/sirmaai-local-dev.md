# Runbook: AgenticSAI Local-Dev Configuration

**Last updated**: 2026-05-14
**Owner**: backend
**Related stories**: S04.21, S04.22, S04.25, S04.31

---

## Purpose

A fresh developer cloning `eusolicit/` for the first time gets a working `make up`
against AgenticSAI staging within 15 minutes, with a clear path to bootstrapping the
shared dev Org/Project credentials.

**Out of scope**: production AgenticSAI deployment (see
[`agenticsai-key-rotation.md`](agenticsai-key-rotation.md) and
[`agenticsai-webhook-ingress.md`](agenticsai-webhook-ingress.md)).

---

## Pre-flight check

Before starting, confirm:

- [ ] Docker is running: `docker info`
- [ ] `eusolicit/` is cloned locally
- [ ] `eusolicit-app/.env.example` exists (the canonical template)

---

## Step-by-step

### 1. Copy the env template

```bash
cp eusolicit-app/.env.example eusolicit-app/.env
```

### 2. Generate a local Fernet key

```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

Paste the output into `.env`, uncommenting the `AGENTICSAI_FERNET_KEY=` line:

```
AGENTICSAI_FERNET_KEY=<your-generated-key>
```

### 3. Obtain shared-dev Org credentials from the project lead

Ask Deb for:
- **Shared-dev Org UUID** → paste into `AGENTICSAI_ORG_ID=<uuid>`
- **Shared-dev Org admin API key** → paste into `AGENTICSAI_ADMIN_API_KEY=<key>`

> These are not committed because the shared-dev Org is shared across all
> developers. Rotating it requires a coordinated re-bootstrap of every dev's
> `.env` — operator-hand-off pattern (same as the Stripe test-mode key for
> Story 15.0).

### 4. Enable the AgenticSAI feature flag

In `.env`, uncomment and set:

```
AGENTICSAI_GATEWAY_ENABLED=true
```

Leave it commented to stay on the pre-amendment AgenticSAI path during the
cutover window.

### 5. (Optional) Set up an ngrok tunnel for webhook receive

Only needed if you plan to test AgenticSAI webhook delivery locally.

```bash
ngrok http 8004
```

Copy the public HTTPS URL and set in `.env`:

```
AGENTICSAI_WEBHOOK_CALLBACK_URL=https://<your-ngrok-id>.ngrok.io/webhooks/agenticsai
```

### 6. Start the full stack

```bash
cd eusolicit-app
make infra          # Start postgres, redis, minio, clamav
make migrate-all    # Run Alembic migrations (required after reset-db)
make up             # Start all services including agenticsai-gateway on :8004
```

### 7. Smoke check

```bash
curl http://localhost:8004/health
# Expected: {"status":"ok"}

curl -s -o /dev/null -w "%{http_code}" http://localhost:8004/ready
# Expected: 200
```

### 8. Admin-bootstrap the shared dev Project (pending future admin-api integration)

> ⚠️ **Status**: The `admin-api` `POST /admin/agenticsai-projects/bootstrap` endpoint
> is not yet implemented (planned for a future story — production per-tenant
> provisioning lives in E24). This step is a placeholder.
>
> When available, the call shape will be:
> ```bash
> curl -X POST http://localhost:8002/admin/agenticsai-projects/bootstrap \
>   -H "Content-Type: application/json" \
>   -d '{
>     "company_id": "<test-company-uuid>",
>     "agenticsai_org_id": "<from-step-3>",
>     "agenticsai_project_id": "<shared-dev-project-uuid>"
>   }'
> ```
>
> Until E24 lands, populate `client.agenticsai_projects` directly via the DB for
> dev/testing, or ask Deb for the current bootstrap procedure.

---

## Failure modes

| Symptom | Root cause | Fix |
|---|---|---|
| `make up` fails with `pydantic.ValidationError: agenticsai_fernet_key` required | `AGENTICSAI_FERNET_KEY` unset while `AGENTICSAI_GATEWAY_ENABLED=true` | Run step 2 above |
| `/admin/circuits` 500s on first call | `AGENTICSAI_ORG_ID` or `AGENTICSAI_ADMIN_API_KEY` unset | Run step 3 above |
| Webhook bootstrap fails with `WEBHOOK_BOOTSTRAP_ENABLED but callback URL empty` | `AGENTICSAI_WEBHOOK_CALLBACK_URL` unset while bootstrap is enabled | Run step 5 above |
| Calls to AgenticSAI hang 60 s+ and return 504 | Default base URL `stage.sirma.ai` unreachable from local network | Verify staging is up; consider VPN; check ADR-019 for current canonical staging URL |
| Env var change silently ignored at boot | `AgenticSAIGatewaySettings` has no `env_prefix` — typos in field names are swallowed by `extra="ignore"` | Double-check env var spelling against field names in `config.py` (field name uppercased = env var) |

---

## Rollback

Comment out the AgenticSAI block in `.env` (`AGENTICSAI_GATEWAY_ENABLED`, `AGENTICSAI_ORG_ID`,
`AGENTICSAI_FERNET_KEY`, etc.) and restart the stack. The flag-off path (legacy AgenticSAI
behaviour per S04.20) is preserved as the runtime fallback.

---

## Related

- [`eusolicit-app/CLAUDE.md`](../../eusolicit-app/CLAUDE.md) — per-app local-stack supplement
- [Project CLAUDE.md](../../CLAUDE.md) — service ports, architecture, commands
- [ADR-019](../planning-artifacts/architecture-amendment-2026-05-12-agenticsai.md) — local-dev decision (Option 1: staging; Option 2: mock deferred to E26+)
- [S04.21 story](../implementation-artifacts/4-21-agenticsai-projects-schema-and-project-cache.md) — `client.agenticsai_projects` schema
- [S04.22 story](../implementation-artifacts/4-22-per-project-api-key-vault-and-rotation.md) — Fernet key vault + rotation worker
- [S04.25 story](../implementation-artifacts/4-25-standard-webhooks-receiver.md) — webhook receiver + HMAC verification
- [S04.31 story](../implementation-artifacts/4-31-local-dev-agenticsai-configuration.md) — this runbook's landing story
- [`agenticsai-key-rotation.md`](agenticsai-key-rotation.md) — production key rotation ops
- [`agenticsai-webhook-ingress.md`](agenticsai-webhook-ingress.md) — production webhook ingress ops
- [`agenticsai-agent-inventory.md`](agenticsai-agent-inventory.md) — agent inventory ops
