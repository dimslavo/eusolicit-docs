# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo layout

Two sibling trees, no shared tooling:

- `sirma-ai-k8s-apps/` — production GitOps source of truth. ArgoCD app-of-apps + a single shared Helm chart (`charts/sirma-ai-service/`) parametrized by `apps/<service>/values-dev.yaml`. No application source code lives here — services are deployed from prebuilt images at `registry.sirma.ai/sirma-ai-{private,public}/<service>:<tag>`.
- `local-deploy/` — self-contained docker-compose translation of those values files. No AWS/Google/Stripe/SES/external-SaaS dependencies. The user actively iterates on this; it is the primary working surface in most sessions.

When something in `local-deploy/` looks wrong, the answer almost always lives in `sirma-ai-k8s-apps/apps/<service>/values-dev.yaml` (env block) or `sirma-ai-k8s-apps/charts/sirma-ai-service/templates/`.

## Local stack (everyday commands)

```bash
cd local-deploy
docker compose up -d
docker compose logs -f sirma-ai-backend     # backend takes the longest to come up
docker compose down                         # keep volumes
docker compose down -v                      # also wipe postgres / minio / weaviate
docker compose restart sirma-ai-<service>   # after editing env vars
```

First boot runs `postgres/init.sql`, `vault/seed.sh`, `minio/init.sh`, then app-side migrations (e.g. `alembic upgrade head` for `sirma-ai-ai`).

The stack has no test suite or build step — it is purely a runtime composition of vendor images. `local-deploy/README.md` has the per-service host-port and credentials table (use it instead of grepping the compose file).

## Per-service databases

`postgres/init.sql` creates one role+database per app: `k8s_sirma_ai_backend`, `k8s_sirma_ai_ai`, `k8s_sirma_ai_storage`, `k8s_sirma_ai_crawler`, `mcp_proxy`. Passwords are `devpass-<svc>`. Three places must agree, or a service will fail to connect:

1. `postgres/init.sql` (creates user + DB)
2. `vault/seed.sh` (writes the same value into `secret/dev/<app>` so backend's runtime Vault reads match)
3. The service's `environment:` block in `docker-compose.yml` (`DB_PASS` / `DBPASS`)

When changing a password, update all three.

## Backend RSA keys

`local-deploy/keys/{private-key.pem,public-key.pem}` is mounted into the backend at `/app/.keys` and referenced via `RSA_PRIVATE_KEY` / `RSA_PUBLIC_KEY` (Spring `file://...` resolver). These are the signing keys for issued tokens; regenerate with `openssl genrsa` + `openssl rsa -pubout` if you ever need a fresh pair.

## How prod → local was translated (mental model)

Apps were written for k8s + Vault + AWS. The compose file fakes that environment without touching app code:

| Prod dependency | Local replacement | Notes |
|---|---|---|
| AWS RDS / pgbouncer | `postgres` with network alias `pgbouncer.sirma-ai` | Apps connect to the alias, not `postgres` |
| Vault HA + ESO `envFrom` | Single dev-mode `vault` (alias `vault.vault.svc.cluster.local`) + secrets injected as docker `environment` directly | ESO doesn't run locally; we pre-render the secrets into env vars |
| AWS S3 | `minio` (`http://minio:9000`, creds `minioadmin/minioadmin`) | `minio-init` creates the buckets |
| AWS SES | `mailhog` aliased as `email-smtp.us-east-1.amazonaws.com`, SMTP on :1025, UI on :8025 |
| OpenAI/Anthropic | `registry.sirma.ai/sirma-ai-public/ai:openai_mock_v1` at `http://openai-mock:9999/v1` |
| Route53 + K8s API (n8n dynamic instances) | **Skipped.** `sirma-ai-n8n-deployment` is intentionally absent from compose |
| Google OAuth, Stripe | placeholder creds, features won't function but apps boot |

The network aliases (`pgbouncer.sirma-ai`, `vault.vault.svc.cluster.local`, `email-smtp.us-east-1.amazonaws.com`, `weaviate-rpc.sirma-ai.svc.cluster.local`, `livekit-redis.sirma-ai.svc.cluster.local`) exist so that env-var values copied verbatim from `values-dev.yaml` resolve without modification.

## Vault and secrets

Prod path layout (KV v2 under `secret/dev/<app>`) is reproduced by `vault/seed.sh` so any app that hits Vault at runtime sees the right shape. **However**, locally the secrets are *also* injected directly via docker `environment:` blocks because no ESO runs to translate Vault → envFrom. Backend is the only app that opens a runtime Vault client (`VAULT_URL`, `VAULT_TOKEN=dev-root-token`).

Per-service Vault paths and which secrets each app expects are documented in `infrastructure/vault/VAULT-ACCESS.md` (in the k8s-apps tree).

## Spring Boot env-var gotcha (backend)

The k8s `values-dev.yaml` env block names (`DBPASSWORD`, `SMTP_PASSWORD`, `STRIPE_APIKEY`, `LIVEKIT_API_SECRET`) are NOT what the JAR reads. The actual placeholders inside `BOOT-INF/classes/application.yaml` are `DBPASS`, `SMTP_PASS`, `STRIPE_SECRET_KEY`, `LIVEKIT_API_KEY_SECRET`. Compose uses the JAR-correct names. To verify what the JAR actually expects:

```bash
docker create --name tmp registry.sirma.ai/sirma-ai-private/backend:<tag>
docker cp tmp:/app/app.jar /tmp/app.jar && docker rm tmp
unzip -p /tmp/app.jar BOOT-INF/classes/application.yaml | less
```

Spring relaxed binding lets env vars override hard-coded properties (e.g. `SPRING_MAIL_PORT=1025` overrides `spring.mail.port: 587`).

AES/HMAC-style keys (`INVITE_TOKEN_AES_KEY`, `INVITE_TOKEN_HMAC_SECRET`, `SERVICE_KEY_*`, `SERVICE_TOKEN_SECRET`) are **base64-decoded** by the app — must be base64-encoded 32-byte values (`openssl rand -base64 32`), not hex or arbitrary strings, or boot fails with `IllegalArgumentException: Illegal base64 character`.

Python services use snake_case (`DB_PASS`, `DB_HOST`); the Java backend uses run-together names (`DBPASS`, `DBHOST`).

## Public hostname (nginx + Let's Encrypt)

Local stack is fronted by the host's system nginx at `agenticsai.endigitalx.com`. Vhost templates live at `local-deploy/nginx/agenticsai.endigitalx.com{,.bootstrap}.conf` and `lk.agenticsai.endigitalx.com{,.bootstrap}.conf`. Pattern (matches existing `*.endigitalx.com` sites on the host):

1. Install HTTP-only `*.bootstrap.conf` first
2. `sudo certbot certonly --webroot -w /var/www/certbot -d <subdomain> ...`
3. Replace with full HTTP→HTTPS vhost
4. `sudo nginx -t && sudo systemctl reload nginx`

LiveKit media (UDP 7882) must be exposed directly — nginx cannot proxy it.

**IP allowlist:** `local-deploy/nginx/allowed-ips.conf` is the single source of truth for who can reach the platform. Both vhosts `include` it in their HTTPS server blocks and HTTP→HTTPS redirect (ACME challenge stays open so Let's Encrypt can renew). LiveKit media ports (UDP 7882 / TCP 7881) bypass nginx entirely — they are NOT covered by this allowlist; restrict at the host firewall if needed.

**The user does not have passwordless sudo configured for Claude.** Anything that touches `/etc/nginx/`, `/etc/letsencrypt/`, or `systemctl` must be returned to the user as commands they run themselves (do not attempt `echo password | sudo -S`).

## Image pinning (k8s-apps tree)

Image tags are owned by `argocd-apps/dev/sirma-ai/image-updater.yaml` — ArgoCD Image Updater writes the resolved digest into `apps/<service>/values-dev.yaml`. **Do not manually edit `image.tag` / `image.repository` in the values files** — image-updater overwrites them on the next sync. To pin or roll back, edit the ImageUpdater CRD instead. The local compose file pins specific image digests/tags directly.

## Things to skip / not over-engineer

- Don't try to recreate ESO, vault-agent sidecars, or the HA Vault deployment locally — they're explicitly excluded.
- Don't add pgbouncer locally — apps connect straight to postgres via the `pgbouncer.sirma-ai` alias.
- The `$(DB_PASS)` substitution pattern in k8s values files is `envFrom`-aware k8s variable expansion. It does NOT work in plain docker — values must be pre-substituted in compose `environment:` blocks.
