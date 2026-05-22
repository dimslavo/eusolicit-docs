# Runbook: AgenticSAI Webhook Ingress (`api.eusolicit.com`)

**Severity**: SEV-2 (public ingress down blocks all AgenticSAI webhook delivery)
**Last updated**: 2026-05-14
**Story**: S04.29 — Public Ingress for Webhook Receiver

---

## Purpose

This runbook covers enabling, operating, and rolling back the public nginx ingress
that routes `https://api.eusolicit.com/webhooks/agenticsai` to the `agenticsai-gateway`
container (port 18004) on www1.

The S04.25 receiver (`POST /webhooks/agenticsai`) implements HMAC-over-raw-bytes
signature verification, 7-day Redis idempotency, and a DLQ table — this runbook
covers the **transport layer only** (nginx + TLS). Application-layer incidents
should start from the S04.25 receiver code and its Prometheus counters.

By design, `api.eusolicit.com` exposes **exactly one path**:
- `POST /webhooks/agenticsai` → `http://127.0.0.1:18004/webhooks/agenticsai`
- Every other URI on `api.eusolicit.com` → `HTTP 404` (explicit catch-all; no upstream)

---

## When to Use

- **First-time enabling AgenticSAI inbound webhooks** — run §Pre-flight → §Deploy.
- **Reverting the public ingress after a security incident** — run §Rollback.
- **DR rebuild Phase 2 cert-issuance step** (in conjunction with
  [`www1-rebuild.md`](www1-rebuild.md)) — add `-d api.eusolicit.com` to the
  certbot command as documented in §Deploy step 4.
- **TLS expiry / cert mismatch incident** on `api.eusolicit.com` — run §Deploy
  step 4 (certbot --expand) and §Deploy step 5 (nginx reload).
- **`nginx -t` syntax error after manual edit on www1** — follow §Failure modes
  row 1 to reconcile `/etc/nginx/sites-available/eusolicit.com` with the repo.

---

## Pre-flight

Run all checks **before** proceeding with §Deploy. A failed pre-flight check means
the deploy will fail mid-run; fix the prerequisite first.

### 1. DNS — A record for `api.eusolicit.com`

```bash
dig +short api.eusolicit.com
```

**Expected**: `46.10.208.159` (the canonical www1 IP).

If the output is empty or wrong, create/fix the A record at the registrar **before
proceeding**. Certbot's HTTP-01 ACME challenge (§Deploy step 4) resolves this
hostname from the public internet — if DNS isn't pointing at www1, the challenge
will fail and certbot will abort, leaving the cert unchanged.

> **Operator action required**: DNS records are out of host-as-code scope
> (per `infra/host/README.md` line 51 — "DNS records (out of host scope; operator
> action)"). This runbook documents the required record; provisioning it at the
> registrar is the operator's responsibility.

### 2. Container health — agenticsai-gateway on port 18004

```bash
ssh debian@www1.endigitalx.com 'curl -sf http://127.0.0.1:18004/healthz'
```

**Expected**: HTTP 200. If not, chain to
[`runbooks/container-restart-loop.md`](container-restart-loop.md) before
proceeding. A healthy container is required for the smoke test (§Deploy step 6).

### 3. AgenticSAI gateway flag

```bash
ssh debian@www1.endigitalx.com \
  'grep AGENTICSAI_GATEWAY_ENABLED /home/debian/eusolicit-overrides/.env.prod'
```

**Expected**: `AGENTICSAI_GATEWAY_ENABLED=true`. If false, the receiver returns 503
and the smoke test will incorrectly appear as an ingress failure. Flip the flag
and redeploy the gateway per the S04.20 cutover plan before running the smoke test.

### 4. Cert SAN check — confirm `api.eusolicit.com` is NOT yet a SAN

```bash
ssh debian@www1.endigitalx.com \
  'sudo openssl x509 \
     -in /etc/letsencrypt/live/www.eusolicit.com/fullchain.pem \
     -noout -text | grep -i "DNS:"'
```

**Expected (first-time run)**: output lists `www.eusolicit.com`, `eusolicit.com`,
`admin.eusolicit.com` — but NOT `api.eusolicit.com`. This confirms §Deploy step 4
(certbot --expand) is still needed.

If `api.eusolicit.com` already appears in the SAN list, the cert has already been
expanded; skip §Deploy step 4 and proceed directly to step 5 (nginx reload).

---

## Deploy

Copy-paste each step verbatim. All commands run on www1 as user `debian`.

```bash
# ── Step 1: SSH to www1 ──────────────────────────────────────────────────────
ssh debian@www1.endigitalx.com
cd /home/debian/Projects/eusolicit/eusolicit-app

# ── Step 2: Pull latest main (includes S04.29 nginx changes) ─────────────────
git pull

# ── Step 3: Sync nginx config to live location ───────────────────────────────
# Save a pre-deploy backup first — required for rollback if the git commit
# has not yet landed (see §Rollback precondition check).
sudo cp /etc/nginx/sites-available/eusolicit.com /tmp/eusolicit.com.bak
sudo cp infra/nginx/eusolicit.com /etc/nginx/sites-available/eusolicit.com
sudo nginx -t
# MUST return:
#   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
#   nginx: configuration file /etc/nginx/nginx.conf test is successful
# If it returns a syntax error, see §Failure modes row 1 before continuing.

# Activate the updated :80 block (required for certbot's ACME challenge in
# step 4). Brief TLS window: api.eusolicit.com :443 is now active but the
# Let's Encrypt cert does NOT yet include the api.eusolicit.com SAN — any TLS
# handshake during steps 3-5 fails cert-name validation. Run steps 4 + 5
# promptly; configure AgenticSAI to deliver webhooks ONLY after step 5 completes.
sudo systemctl reload nginx

# ── Step 4: Extend cert SAN (one-time — first enable of api.eusolicit.com) ───
# Uses webroot mode (not --nginx) to prevent certbot from rewriting the nginx
# config in-place, which would cause drift from the repo file on the next
# sudo cp. The :80 block (reloaded in step 3) serves the ACME challenge at
# /.well-known/acme-challenge/ from /var/www/certbot.
# Only run this step once. Subsequent cert renewals happen automatically via
# the certbot systemd timer because the SAN is now part of the cert.
sudo certbot certonly --webroot -w /var/www/certbot --expand \
  -d www.eusolicit.com \
  -d eusolicit.com \
  -d admin.eusolicit.com \
  -d api.eusolicit.com
# Certbot will perform an HTTP-01 ACME challenge for api.eusolicit.com.
# DNS must be propagated (§Pre-flight check 1) or this step will fail.
# If it fails, the previous cert remains valid — nginx is unaffected.
# Wait for DNS propagation and retry this step.

# ── Step 5: Reload nginx (new cert takes effect — TLS window closed) ─────────
sudo systemctl reload nginx

# ── Step 6: Smoke test — verify the webhook path is reachable ────────────────
curl -sv https://api.eusolicit.com/webhooks/agenticsai \
  -X POST \
  -H 'webhook-id: smoke-test-12345' \
  -H "webhook-timestamp: $(date +%s)" \
  -H 'webhook-signature: v1,deadbeef' \
  -H 'Content-Type: application/json' \
  -d '{"type":"smoke.test"}'
# Expected: HTTP 401 {"error":"no_active_subscription"} or {"error":"invalid_signature"}
# Both responses prove:
#   (a) nginx terminated TLS and routed to the gateway
#   (b) the gateway parsed the request headers
#   (c) S04.25's signature verification fired
# A 503 response means AGENTICSAI_GATEWAY_ENABLED=false in .env.prod, OR the env
# var was changed but the container was not restarted (env is loaded at
# container startup, not live-refreshed). Diagnostic: confirm §Pre-flight
# check 3 flag is true, then restart:
#   docker compose -f docker-compose.prod.yml restart agenticsai-gateway
# A 502/504 or connection-refused means nginx cannot reach 127.0.0.1:18004
# — debug per §Failure modes row 3.

# ── Step 7: Verify catch-all 404 blocks all other paths ──────────────────────
curl -sv https://api.eusolicit.com/api/v1/agents/foo/run
curl -sv https://api.eusolicit.com/admin/circuits
curl -sv https://api.eusolicit.com/healthz
# Expected for all three: HTTP 404
# (the explicit catch-all `location /` from AC 1 — no upstream is called)

# ── Step 8: Verify the old /ai/ surface on www is closed ─────────────────────
curl -sv https://www.eusolicit.com/ai/healthz
# Expected: HTTP 404 (was previously 200 before S04.29 — confirms AC 3 deletion is live)
```

> **Safe to re-run**: Steps 2, 3, 5, 6, 7, 8 are idempotent. `cp` overwrites,
> `nginx -t` is read-only, `systemctl reload` is graceful. Step 4 (certbot
> certonly --webroot --expand) is idempotent only if the SAN is already
> present; if the cert already has `api.eusolicit.com`, certbot exits cleanly
> with "Certificate not yet due for renewal". No harm in re-running.

---

## Verify End-to-End (Operator Action — AgenticSAI Admin UI)

After completing §Deploy, verify the full path from AgenticSAI's delivery agent:

1. Log into the AgenticSAI admin console at `agenticsai.endigitalx.com`.
2. Navigate to the webhook configuration for the EU Solicit subscription.
3. Trigger a test delivery pointed at `https://api.eusolicit.com/webhooks/agenticsai`.
4. **Expected**: HTTP 200 response from the receiver + a row inserted in
   `gateway.webhook_log` (visible via the admin API or psql).
5. Verify the test delivery's `webhook-id` appears in the Redis idempotency cache:
   ```bash
   ssh debian@www1.endigitalx.com \
     'docker compose -f docker-compose.prod.yml exec redis \
        redis-cli GET "agenticsai:webhook:dedup:<webhook-id>"'
   ```
   **Expected**: `1` (the idempotency key set by S04.25's dedup logic).

---

## Rollback

The rollback reverts the nginx configuration change. **The cert SAN is not
rolled back** — removing a SAN requires re-issuing the cert without `--expand`
(a separate operator task not covered here). Leaving the SAN in place is
harmless if DNS for `api.eusolicit.com` is kept live; the certbot auto-renew
timer will keep refreshing it.

**Precondition**: First check whether the S04.29 commit is in git history:

```bash
git log --oneline -- infra/nginx/eusolicit.com | head -3
```

If the S04.29 commit appears as the most-recent entry, use the git-based rollback below.

If the S04.29 commit does **NOT** appear (rolling back during the deploy window before the commit is merged), restore from the pre-deploy backup captured in §Deploy step 3:

```bash
sudo cp /tmp/eusolicit.com.bak /etc/nginx/sites-available/eusolicit.com
sudo nginx -t && sudo systemctl reload nginx
```

**Git-based rollback** (when S04.29 commit is confirmed in history):

```bash
# On www1
cd /home/debian/Projects/eusolicit/eusolicit-app

# Find the commit before the S04.29 change:
git log --oneline -- infra/nginx/eusolicit.com | head -5
# Check out the pre-S04.29 nginx file from git history:
git show <commit-before-S04.29>:eusolicit-app/infra/nginx/eusolicit.com \
  > /tmp/eusolicit.com.pre-s04.29

# Sync the reverted config to nginx:
sudo cp /tmp/eusolicit.com.pre-s04.29 /etc/nginx/sites-available/eusolicit.com
sudo nginx -t && sudo systemctl reload nginx
```

After rollback, `https://api.eusolicit.com/webhooks/agenticsai` will return 502 (no
nginx server block for that host). The cert SAN stays — harmless unless the
operator explicitly decides to remove it (non-trivial, not covered here).

---

## Failure Modes

| Symptom | Likely cause | Resolution |
|---|---|---|
| `nginx -t` syntax error after `sudo cp` | Hand-edit drift in `/etc/nginx` | `diff /etc/nginx/sites-available/eusolicit.com infra/nginx/eusolicit.com` to find divergence; re-cp from repo |
| `certbot --expand` fails ACME challenge | DNS for `api.eusolicit.com` not propagated | `dig +short api.eusolicit.com` from a third-party resolver; wait + retry. Cert is unaffected — old cert still valid during wait |
| `curl https://api.eusolicit.com/webhooks/agenticsai` → 502 | `agenticsai-gateway` container down on port 18004 | `docker compose -f docker-compose.prod.yml ps agenticsai-gateway`; chain to [`runbooks/container-restart-loop.md`](container-restart-loop.md) |
| `curl https://api.eusolicit.com/webhooks/agenticsai` → 503 | `AGENTICSAI_GATEWAY_ENABLED=false` in `.env.prod` | Flip flag, redeploy gateway per S04.20 cutover plan |
| TLS `unknown CA` / cert mismatch on `api.eusolicit.com` | SAN expansion didn't complete | Re-run `certbot certonly --webroot -w /var/www/certbot --expand` with all four `-d` flags; `sudo systemctl reload nginx` |
| `curl https://api.eusolicit.com/webhooks/agenticsai` → 404 | Server block not reloaded, or wrong `server_name` | `sudo nginx -t` → confirm `api.eusolicit.com` block is present; `sudo systemctl reload nginx` |
| AgenticSAI delivery reports 200 but no row in `gateway.webhook_log` | Application-layer HMAC failure before DB write | Check agenticsai-gateway logs; S04.25 receiver may be logging `invalid_signature` — verify AgenticSAI webhook secret matches `AGENTICSAI_WEBHOOK_SECRET` in `.env.prod` |
| `docker compose exec` fails on Redis verify | Redis container not running | `docker compose -f docker-compose.prod.yml up -d redis`; chain to [`runbooks/docker-daemon-recovery.md`](docker-daemon-recovery.md) |

---

## References

- **Story**: [`eusolicit-docs/implementation-artifacts/4-29-public-ingress-for-webhook-receiver.md`](../implementation-artifacts/4-29-public-ingress-for-webhook-receiver.md)
- **Receiver story (S04.25)**: [`eusolicit-docs/implementation-artifacts/4-25-standard-webhooks-receiver.md`](../implementation-artifacts/4-25-standard-webhooks-receiver.md)
- **Epic definition (S04.29 + AC line 485)**: [`eusolicit-docs/planning-artifacts/epics/E04-agenticsai-gateway-service.md`](../planning-artifacts/epics/E04-agenticsai-gateway-service.md)
- **Architecture amendment §3.4 + §5.1**: [`eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-agenticsai.md`](../planning-artifacts/architecture-amendment-2026-05-12-agenticsai.md)
- **DR rebuild runbook**: [`eusolicit-docs/runbooks/www1-rebuild.md`](www1-rebuild.md)
- **AgenticSAI key rotation runbook**: [`eusolicit-docs/runbooks/agenticsai-key-rotation.md`](agenticsai-key-rotation.md)
- **Container restart loop**: [`eusolicit-docs/runbooks/container-restart-loop.md`](container-restart-loop.md)
- **Docker daemon recovery**: [`eusolicit-docs/runbooks/docker-daemon-recovery.md`](docker-daemon-recovery.md)
- **Deploy manual discipline**: project memory `project_deploy_nginx_manual.md`
- **www1 host state (IP, paths)**: project memory `project_www1_host_state.md`
