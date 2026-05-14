# Story 4.29: Public Ingress for Webhook Receiver

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **DevOps engineer landing the publicly-reachable surface that SirmaAI's Standard Webhooks delivery agent will POST to (Concern #4 from `implementation-readiness-report-2026-05-12-sirmaai.md` line 247 — "Original E04 AC #15 reads ClusterIP-only; the webhook receiver MUST be publicly reachable for SirmaAI to deliver Standard Webhooks") + Epic 4 amendment AC line 485 + line 511 ("**Public ingress configured for webhook receiver only (path `/webhooks/sirmaai`); all other gateway paths remain ClusterIP-only** … nginx vhost on www1 routing `https://api.eusolicit.com/webhooks/sirmaai` → cluster-internal `sirmaai-gateway:8004/webhooks/sirmaai`; **all other gateway paths remain ClusterIP-only**. Requires manual sudo cp on www1 per `project_deploy_nginx_manual.md` (memory). TLS via existing certbot. Runbook entry at `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`. Closes readiness Concern #4")**,

I want **(a) a NEW nginx server block for `api.eusolicit.com` added to BOTH `infra/nginx/eusolicit.com` (the canonical repo reference copy that `project_deploy_nginx_manual.md` mandates be `sudo cp`'d into `/etc/nginx/sites-available/eusolicit.com` on www1 by hand) AND `infra/host/templates/nginx-sites/eusolicit.com.conf.j2` (the Ansible template that the disaster-recovery `infra/host/playbooks/03-nginx-and-certbot.yml` re-deploys on a www1 rebuild per Story onprem-06) — the two files MUST stay byte-identical apart from the Jinja header so a `www1-rebuild.md` Phase 1 + Phase 2 sequence reconstructs the exact same nginx state that production runs; (b) the new server block listens on `:443 ssl` with `server_name api.eusolicit.com` and serves **exactly one** `location` — `/webhooks/sirmaai` (literal path, NOT `~ ^/webhooks/sirmaai`) — proxying to `http://127.0.0.1:18004/webhooks/sirmaai` (the host-published port of the `sirmaai-gateway` container per `docker-compose.prod.yml` line 267 `127.0.0.1:18004:8004` + the `1`+container-port host-port convention encoded in the repo nginx file lines 79-90 — DO NOT use the docker-network hostname `sirmaai-gateway` because nginx runs on the host network namespace and cannot resolve docker-internal DNS; the host-port allocation IS the contract between the docker stack and nginx); (c) **all other URIs on `api.eusolicit.com` return HTTP 404 via an explicit catch-all `location /` returning `return 404;`** (no `proxy_pass`, no upstream call, no body leak — the absence of a fallback would default to nginx's autoindex/default-page behavior; the explicit 404 closes the only public surface to the single sanctioned path); (d) a paired `:80` HTTP→HTTPS redirect server block for `api.eusolicit.com` that ALSO serves `/.well-known/acme-challenge/` from `/var/www/certbot` (mirroring the existing `eusolicit.com`/`www.eusolicit.com`/`admin.eusolicit.com` block at file lines 14-26) so certbot's standalone HTTP-01 ACME challenge can renew the `api.eusolicit.com` SAN without a TLS dance — the existing certbot systemd timer on www1 picks this up automatically once the SAN is included in the cert; (e) the new `api.eusolicit.com` host MUST be added to the existing letsencrypt certificate via `certbot --nginx --expand -d www.eusolicit.com -d eusolicit.com -d admin.eusolicit.com -d api.eusolicit.com` (one operator-issued command on www1, captured in the new runbook) — the existing cert at `/etc/letsencrypt/live/www.eusolicit.com/fullchain.pem` (which the existing server blocks already reference) gets the new SAN added in-place and the existing `ssl_certificate` line on the new `api.eusolicit.com` server block points at the SAME pem file (NEVER a separate `/etc/letsencrypt/live/api.eusolicit.com/` chain — that would require dual-cert juggling for renewal); (f) on the `/webhooks/sirmaai` location: explicit `client_max_body_size 1m;` cap (SirmaAI Standard Webhooks payloads are kilobytes; a 1MB ceiling caps DoS payload-flood attack surface while leaving 100× headroom over the largest plausible legitimate event — the `client.sirmaai_kb_files.sha256` writeback events are the upper bound and they are tiny), explicit `proxy_read_timeout 30s` (the webhook handler completes in <100ms p95 per S04.25 contract — 30s is 300× the budget; anything taking 30s is a deadlocked handler that should fail-fast back to SirmaAI, which will retry per Standard Webhooks discipline), explicit `proxy_request_buffering on;` (the default — affirms the full body is buffered before forwarding so the upstream handler sees the body in a single `await request.body()` call without surprise streaming; S04.25 AC 1 step 6 reads the raw bytes once and parses JSON from them, so a streamed body would break the signature-over-raw-bytes contract); (g) standard `include snippets/proxy-params.conf;` for X-Forwarded-* propagation (the snippet was deployed by `03-nginx-and-certbot.yml` lines 22-37 and is the single source of truth for proxy headers — DO NOT inline the `proxy_set_header` directives, that would diverge from the other server blocks); (h) `include snippets/security-headers.conf;` (the existing snippet at `03-nginx-and-certbot.yml` lines 39-50: X-Frame-Options/X-Content-Type-Options/Referrer-Policy/X-XSS-Protection — defense-in-depth on a public surface that returns JSON only, but the headers are cheap and consistent across the vhost family); (i) **CRITICAL audit of the existing `/ai/` location at `infra/nginx/eusolicit.com` lines 132-137**: that block currently proxies `https://www.eusolicit.com/ai/*` → `http://127.0.0.1:18004/*` — meaning **every** gateway endpoint (admin routes, agent-run routes, SSE proxy, the webhook receiver itself) is reachable publicly today through the `www.eusolicit.com/ai/` prefix. Per Epic 4 amendment AC line 485 (\"**all other gateway paths remain ClusterIP-only**\"), this story DELETES the existing `/ai/` location block from BOTH nginx files so that — after this story lands — the ONLY public gateway path is `https://api.eusolicit.com/webhooks/sirmaai`. Confirm no client-api, admin-api, or frontend caller depends on the `/ai/` prefix (grep `services/` + `frontend/` for `"/ai/"` and `eusolicit.com/ai` — all gateway calls go through internal docker-network hostname `sirmaai-gateway:8004` or its `ai-gateway` alias, never through the public www host; the `/ai/` block is dead public surface left over from a deferred-but-never-arrived plan). The deletion is INSIDE this story's scope (single atomic change set) because leaving it in place after S04.29 lands would violate the Epic 4 amendment AC verbatim and re-open Concern #4 from the opposite end; (j) **runbook at `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`** (NEW file, mirroring the existing `runbooks/www1-rebuild.md` + `runbooks/sirmaai-key-rotation.md` format) covering: pre-flight check (DNS A record for `api.eusolicit.com` resolves to `46.10.208.159` per memory `project_www1_host_state.md`, sirmaai-gateway container is running and `127.0.0.1:18004/healthz` returns 200, current cert does NOT yet cover api.eusolicit.com SAN), deploy steps (SSH to www1, `git pull`, `sudo cp infra/nginx/eusolicit.com /etc/nginx/sites-available/eusolicit.com`, `sudo certbot --nginx --expand -d www.eusolicit.com -d eusolicit.com -d admin.eusolicit.com -d api.eusolicit.com`, `sudo nginx -t`, `sudo systemctl reload nginx`, end-to-end smoke `curl -sf https://api.eusolicit.com/webhooks/sirmaai -X POST -H 'webhook-id: smoke-test' …` returns 400 missing-headers or 401 invalid-signature — both prove the path is reachable and the upstream IS the gateway), rollback (revert the nginx file to pre-S04.29 state, `sudo cp` again, `sudo nginx -t && sudo systemctl reload nginx`; cert SAN is NOT rolled back because removing a SAN from an existing cert requires re-issuance and is non-trivial — leaving the SAN in place is harmless if DNS for api.eusolicit.com is left up), DNS prerequisite (operator must create A record for `api.eusolicit.com` → www1 IP **BEFORE** running certbot, otherwise the HTTP-01 challenge fails; documented as an operator task in §Pre-flight); (k) **DNS provisioning is documented as an operator task in the runbook but is NOT automated by this story** (DNS records are out of host-as-code scope per `infra/host/README.md` line 51 "DNS records (out of host scope; operator action)") — the story DOES specify the A record's target IP (46.10.208.159, the canonical www1 IP per memory `project_www1_host_state.md`) and the exact `dig` verification command, but the actual provider-side DNS update is human-only; (l) **no application-code changes** — the `POST /webhooks/sirmaai` endpoint already exists in `services/sirmaai-gateway/src/sirmaai_gateway/routers/webhooks.py` per S04.25 (status: done), the application-layer signature verification + replay-window + idempotency cache + DLQ are the ONLY defense against unauthenticated payloads (no IP allowlist on the public path — Standard Webhooks discipline + S04.25 design); the gateway service does NOT need restart because nginx-only changes don't touch container state — `sudo systemctl reload nginx` is sufficient (this is the runbook §Deploy step's discipline); (m) **GitHub Actions deploy.yml is NOT changed** — per memory `project_deploy_nginx_manual.md`, `scripts/deploy.sh` never touches `/etc/nginx` and that property is preserved by this story; the nginx change lands as a code commit on main (so the canonical repo state matches what's on www1), the deploy workflow ships the code unchanged, and the operator runs the manual SSH steps separately per the new runbook (this is the documented EU-Solicit on-prem pattern, not a workaround); (n) **observability touch-point**: nginx access log entries for `api.eusolicit.com` flow into the existing `/var/log/nginx/eusolicit.access.log` shared by the other vhosts (line 51 of the current site config sets `access_log /var/log/nginx/eusolicit.access.log main;` — the new `api.eusolicit.com` server block inherits this by not overriding `access_log`, which means the existing log-rotation cron + the existing log-aggregation tooling pick up the new traffic for free; per-request `$host` is in the `main` format so api-vs-www can be filtered with `grep 'api.eusolicit.com' /var/log/nginx/eusolicit.access.log`); (o) **NO ufw firewall changes** required — `:80` and `:443` are already open per `infra/host/playbooks/05-firewall.yml`; the new server block adds a vhost on the same already-open ports, not a new listener; (p) **NO docker-compose changes** required — `sirmaai-gateway` (rendered as `ai-gateway` until S04.20/S04.30 service-rename plumbing reaches docker-compose.prod.yml; the network alias `ai-gateway` per CLAUDE.md remains the host-port label) already binds `127.0.0.1:18004:8004`; we just plumb nginx at it; (q) **idempotency at the runbook level**: re-running the deploy step (`sudo cp` again + `sudo nginx -t` + reload) produces no diff if the source file hasn't changed — `cp` is overwrite-by-default, `nginx -t` is read-only, reload is graceful — captured as an explicit "safe to re-run" note in the runbook; (r) **failure modes captured in the runbook**: `nginx -t` reports syntax error → revert + reload; cert renewal fails because DNS not yet propagated → wait + retry the certbot --expand step (the SAN add is the only step that touches certbot, the renew timer is unaffected; if certbot fails partway through the --expand it leaves the previous cert in place and nginx keeps serving — the runbook documents the recovery path explicitly so an operator doesn't panic-reissue); container 18004 not reachable → debug the docker stack first (the runbook chains to `runbooks/container-restart-loop.md` and `runbooks/docker-daemon-recovery.md` as the canonical next-hop runbooks); SirmaAI-side webhook delivery proven via SirmaAI admin UI replay button (operator action, runbook step §Verify-end-to-end)**,

so that **(1) SirmaAI's outbound webhook delivery agent (running on `agenticsai.endigitalx.com` per CLAUDE.md memory `reference_sirmaai_api_docs.md`) can POST to `https://api.eusolicit.com/webhooks/sirmaai` and reach the S04.25 receiver — closes the architecture-amendment §3.4 + §5.1 inbound webhook contract end-to-end (without this story, S04.25's elegant signature-verification + idempotency + DLQ machinery has nobody to receive from in prod); (2) Concern #4 from the SirmaAI pivot's implementation-readiness gate (`implementation-readiness-report-2026-05-12-sirmaai.md` line 247 + line 414) flips from ❌ GAP to ✅ Covered, and the Epic 4 amendment AC line 485 ("Public ingress configured for webhook receiver only") becomes implementation, not aspiration; (3) the §4.4 architectural invariant ("Reconciler is authoritative; webhooks are latency optimisations") survives the public-facing layer untouched — webhooks remain best-effort latency optimisations even when the public ingress is up, because S04.26's reconciler is independent of webhook delivery success; (4) the **on-prem-pivot-from-2026-05-11** discipline (per memory `project_onprem_pivot_2026_05_11.md`: "ADR-010 rewrite: AWS abandoned, single-host Docker on www1 for launch; do not propose AWS managed services without ADR amendment") gets its first new public surface implemented on the chosen architecture rather than retrofitted later; the choice of nginx vhost on the on-prem host (vs. a managed ALB on AWS that we no longer have) is the **correct** boring-tech choice — the runbook captures the manual sudo cp ritual once, in one place, as the operator-facing single source of truth; (5) **all other gateway paths remain ClusterIP-only after this story** — the `/ai/` location deletion in (i) closes the pre-existing public surface that contradicted Epic 4 AC #15 wording from day one; future stories (E04 amendment S04.30 package rename, E26 N8N workflow templates calling webhooks, E28 webhook+reconciliation epic) inherit a clean security posture (one public path, one defense layer, one runbook); (6) the `www1-rebuild.md` Phase 1+2 disaster-recovery sequence remains correct end-to-end — Phase 1 (`ansible-playbook -i inventory.yaml playbooks/site.yml`) re-deploys the Ansible template which now includes the api.eusolicit.com block, Phase 2 (first-run certbot) issues a cert with the api.eusolicit.com SAN (the runbook will get the SAN added to its certbot example invocation so the rebuild ships the new vhost without operator memory); (7) E26 (agent-driven ingestion via N8N workflow templates) has a known stable URL to write back through — N8N workflows running inside SirmaAI's hosted N8N can POST to `https://api.eusolicit.com/webhooks/sirmaai` with the same Standard Webhooks signing that SirmaAI's own delivery agent uses (the receiver is contract-correct regardless of producer); (8) the `eusolicit-docs/runbooks/` library gains one more well-scoped runbook (mirroring the existing `sirmaai-key-rotation.md` + `kraftdata-outage.md` + `www1-rebuild.md` discipline of "one runbook per operator-facing surface"), reducing the on-call engineer's cognitive load when an api.eusolicit.com TLS expiry or 502 incident happens at 3am; (9) **NO new env vars, NO new database migrations, NO new Celery tasks, NO new Prometheus metrics** — this is a pure infrastructure-as-code change with three artifacts (two nginx files + one runbook), three operator steps (DNS + sudo cp + certbot --expand), and zero application-code touchpoints; the change set is the minimal sanctioned response to Concern #4**.

## Acceptance Criteria

1. **NEW server block for `api.eusolicit.com` added to `eusolicit-app/infra/nginx/eusolicit.com`** (the canonical repo reference copy that `project_deploy_nginx_manual.md` mandates be sudo-cp'd to `/etc/nginx/sites-available/eusolicit.com` on www1):

   - The new server block listens on **`:443 ssl`** and `http2 on;` (matching the existing `www.eusolicit.com` server block at file lines 63-66 — consistency is the design principle).
   - **`server_name api.eusolicit.com;`** — exactly one server name, NOT a wildcard, NOT an alias of `www.eusolicit.com`.
   - **`ssl_certificate /etc/letsencrypt/live/www.eusolicit.com/fullchain.pem;`** and **`ssl_certificate_key /etc/letsencrypt/live/www.eusolicit.com/privkey.pem;`** — pointing at the SAME pem chain used by the other vhosts (the api.eusolicit.com SAN gets added to that one cert via `certbot --expand`; do NOT create a separate `/etc/letsencrypt/live/api.eusolicit.com/` chain).
   - **`include snippets/ssl-params.conf;`** and **`include snippets/security-headers.conf;`** — same snippet includes the existing `www.eusolicit.com` block uses (file lines 71-72). DO NOT inline TLS or header directives.
   - **`access_log /var/log/nginx/eusolicit.access.log main;`** — shared with the other vhosts so the existing log-rotation cron + log-aggregation tooling pick up the new traffic.
   - **`error_log /var/log/nginx/eusolicit.error.log warn;`** — same.
   - **`client_max_body_size 1m;`** — explicit 1 MB cap on this server block (NOT inherited from the `www.eusolicit.com` 50M block). Rationale: SirmaAI webhook payloads are kilobytes; 1 MB is 100× headroom over the largest plausible legitimate event and the smallest cap that doesn't risk false-positive rejection on a future payload-shape change. The 50M cap on www would let a malicious actor flood the gateway with megabyte payloads through `api.eusolicit.com` if we forgot this.
   - **One `location = /webhooks/sirmaai`** block (literal `=` match — fastest dispatch, exact match only; NOT `~ ^/webhooks/sirmaai` regex match which is slower and could surprise on path-suffix attacks). Inside the location:
     - `proxy_pass http://127.0.0.1:18004/webhooks/sirmaai;` (with the explicit path suffix on the upstream — NO trailing-slash strip; the upstream receiver expects `/webhooks/sirmaai` and we are NOT collapsing or rewriting). The host port `18004` is the canonical sirmaai-gateway port per `docker-compose.prod.yml` line 267 + the `1`+container-port host convention.
     - `include snippets/proxy-params.conf;` (single source of truth for X-Forwarded-* headers — set by `03-nginx-and-certbot.yml` lines 22-37; DO NOT inline `proxy_set_header` directives, that would diverge from the rest of the file).
     - `proxy_read_timeout 30s;` (300× the S04.25 p95 handler latency budget).
     - `proxy_request_buffering on;` (explicit affirmation — this IS the default but encoding it documents that the upstream needs the full body in one `request.body()` call for S04.25's HMAC-over-raw-bytes contract).
     - **NO** `add_header` directives inside the location — the server-level `security-headers.conf` include carries them.
   - **One `location /` catch-all** that returns `return 404;` — explicit deny on every URI except `/webhooks/sirmaai`. NO `proxy_pass`, NO upstream call, NO body in the 404 response.
   - The new `:443 ssl` server block is inserted **after** the existing `www.eusolicit.com` block (file lines 62-174) — append-at-end placement keeps the file's logical ordering (HTTP→HTTPS redirects first, then HTTPS vhosts ordered alphabetically by their leftmost subdomain: admin → api → www → eusolicit.com → www.eusolicit.com). The new block goes between the existing `eusolicit.com → www` block and the `admin.eusolicit.com → www` block, OR appended at the end of the file — either placement is valid; the canonical choice for this story is **appended at the end of the file** so the diff is a pure append, easier to review and reasoning.

2. **NEW `:80` HTTP→HTTPS redirect block for `api.eusolicit.com`** added to `eusolicit-app/infra/nginx/eusolicit.com`:

   - Option A (preferred): expand the **existing** `:80` server block at file lines 14-26 by adding `api.eusolicit.com` to its `server_name` line. The existing block already serves `/.well-known/acme-challenge/` from `/var/www/certbot` and redirects everything else to `https://$server_name$request_uri` — adding `api.eusolicit.com` to that server_name list gives the new subdomain the same behavior for free (HTTP→HTTPS redirect + ACME challenge served).
   - Option B (rejected): a separate `:80` block JUST for `api.eusolicit.com` — this would duplicate the ACME challenge boilerplate and risk drift between the two blocks. Option A is the choice.
   - Resulting line: `server_name www.eusolicit.com eusolicit.com admin.eusolicit.com api.eusolicit.com;`

3. **DELETE the existing `/ai/` location block** at `eusolicit-app/infra/nginx/eusolicit.com` lines 132-137 (the block proxying `https://www.eusolicit.com/ai/*` → `http://127.0.0.1:18004/*`):

   - Rationale: that block exposes every sirmaai-gateway endpoint publicly through the `www` host, directly contradicting Epic 4 amendment AC line 485 ("**all other gateway paths remain ClusterIP-only**"). Leaving it in place after S04.29 lands would re-open Concern #4 from the opposite end.
   - **Pre-deletion grep audit** (encode in the story task list as a hard prerequisite — must pass with zero hits before the line is deleted):
     - `grep -rn '/ai/' /home/debian/Projects/eusolicit/eusolicit-app/services/ /home/debian/Projects/eusolicit/eusolicit-app/frontend/` — expected: zero hits referring to the `eusolicit.com/ai/` public path. Any internal `/ai/` references are docker-network hostnames (`sirmaai-gateway:8004` / `ai-gateway:8004`) and don't depend on the nginx prefix.
     - `grep -rn 'eusolicit.com/ai' /home/debian/Projects/eusolicit/` — expected: zero hits (the prefix is pure infrastructure-leftover, no caller uses it).
     - If either grep returns a hit, **do NOT delete the block** in this story — file a follow-up story to migrate the caller to the internal docker-network hostname first, and escalate via the bmad-validate-story gate.
   - The deletion is a pure subtraction — no replacement block, no rewrite, no comment placeholder. The file becomes shorter by one location block.

4. **NEW Ansible template at `eusolicit-app/infra/host/templates/nginx-sites/eusolicit.com.conf.j2`** updated to mirror the changes from AC 1+2+3 byte-for-byte (apart from the Jinja header comment):

   - The existing file at `infra/host/templates/nginx-sites/eusolicit.com.conf.j2` is the source-of-truth for the `infra/host/playbooks/03-nginx-and-certbot.yml` Ansible deployment used by the `www1-rebuild.md` disaster-recovery flow (Phase 1, line 33: `ansible-playbook -i inventory.yaml playbooks/site.yml`). Per Story onprem-06 the two nginx files (the canonical `infra/nginx/eusolicit.com` AND the Ansible template) MUST stay in lockstep so a rebuild reconstructs the production state exactly.
   - Apply the same three changes:
     - Append the new `api.eusolicit.com` `:443 ssl` server block (with the `/webhooks/sirmaai` location + the `/` catch-all 404).
     - Extend the existing `:80` server_name line to include `api.eusolicit.com`.
     - Delete the existing `/ai/` location block (file currently lines 132-137 of `eusolicit.com.conf.j2` — actually the .j2 file currently has NO `/ai/` block per inspection because the .j2 template is a slimmer version of the canonical file; the .j2 ends at the `/admin-api/` block. Confirm by re-reading the file before editing — if the .j2 has no `/ai/` block, AC 3 + the AC 4 deletion-mirror reduces to a no-op on the .j2 file).
   - Ensure the Jinja header comment at the top of the .j2 file remains accurate: "Mirrors the live `/etc/nginx/sites-enabled/eusolicit.com` on www1." That comment is the contract that justifies AC 4's existence.

5. **NEW runbook at `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`** following the structure of the existing `eusolicit-docs/runbooks/sirmaai-key-rotation.md` + `eusolicit-docs/runbooks/www1-rebuild.md`:

   - **§Purpose**: one-paragraph statement of what this runbook covers (publishing or rolling back the `api.eusolicit.com` public ingress to sirmaai-gateway's webhook receiver).
   - **§When to use**: bullet list — "First-time enabling SirmaAI inbound webhooks", "Reverting the public ingress after a security incident", "DR rebuild Phase 2 cert-issuance step (in conjunction with `www1-rebuild.md`)".
   - **§Pre-flight**:
     - DNS verification: `dig +short api.eusolicit.com` returns `46.10.208.159` (the canonical www1 IP per memory `project_www1_host_state.md`). If empty, the operator must add an A record at the registrar BEFORE proceeding.
     - Container health: `ssh debian@www1.endigitalx.com 'curl -sf http://127.0.0.1:18004/healthz'` returns 200. If not, chain to `runbooks/container-restart-loop.md`.
     - SirmaAI gateway flag-on: `ssh debian@www1.endigitalx.com 'grep SIRMAAI_GATEWAY_ENABLED /home/debian/eusolicit-overrides/.env.prod'` returns `SIRMAAI_GATEWAY_ENABLED=true` (per S04.20 the flag must be on for the receiver to accept events; if false, the receiver returns 503 and the smoke test will incorrectly read as ingress-failure).
     - Cert SAN check: `ssh debian@www1.endigitalx.com 'sudo openssl x509 -in /etc/letsencrypt/live/www.eusolicit.com/fullchain.pem -noout -text | grep -i "DNS:"'` — confirm `api.eusolicit.com` is NOT in the SAN list yet (this runbook will add it).
   - **§Deploy** (step-by-step, copy-pasteable):
     ```bash
     # 1. SSH to www1
     ssh debian@www1.endigitalx.com
     cd /home/debian/Projects/eusolicit/eusolicit-app
     
     # 2. Pull latest main (which includes the S04.29 nginx changes)
     git pull
     
     # 3. Sync nginx config to live location
     sudo cp infra/nginx/eusolicit.com /etc/nginx/sites-available/eusolicit.com
     sudo nginx -t          # MUST return "syntax is ok" + "test is successful"
     
     # 4. Extend cert SAN (only needed once — the first time api.eusolicit.com
     #    is added; subsequent renewals are automatic via the certbot systemd
     #    timer because the SAN is now part of the cert).
     sudo certbot --nginx --expand \
       -d www.eusolicit.com \
       -d eusolicit.com \
       -d admin.eusolicit.com \
       -d api.eusolicit.com
     
     # 5. Reload nginx so the new server block + new cert take effect
     sudo systemctl reload nginx
     
     # 6. Verify the public path is reachable AND the upstream IS the gateway
     curl -sv https://api.eusolicit.com/webhooks/sirmaai \
       -X POST \
       -H 'webhook-id: smoke-test-12345' \
       -H 'webhook-timestamp: 1715731200' \
       -H 'webhook-signature: v1,deadbeef' \
       -H 'Content-Type: application/json' \
       -d '{"type":"smoke.test"}'
     # Expected: HTTP 401 {"error":"no_active_subscription"} or 401 {"error":"invalid_signature"}
     # — both prove that nginx routed to the gateway, the gateway parsed the
     #   request, and S04.25's signature verification fired. A 502/504/connection-
     #   refused means nginx can't reach the upstream; debug per §Failure modes.
     
     # 7. Verify NO other path on api.eusolicit.com works
     curl -sv https://api.eusolicit.com/api/v1/agents/foo/run
     curl -sv https://api.eusolicit.com/admin/circuits
     curl -sv https://api.eusolicit.com/healthz
     # Expected for all three: HTTP 404 (the catch-all `location /` from AC 1).
     
     # 8. Verify the old /ai/ surface on www is gone
     curl -sv https://www.eusolicit.com/ai/healthz
     # Expected: HTTP 404 (was previously 200 — confirms AC 3 deletion is live).
     ```
   - **§Verify-end-to-end** (operator action — requires SirmaAI admin UI access):
     - Trigger a test webhook delivery from the SirmaAI admin console pointed at `https://api.eusolicit.com/webhooks/sirmaai`. Expected: HTTP 200 + a row in `gateway.webhook_log` (per S04.25 audit + the existing `webhook_log` table from S04.07 carry-forward).
     - Verify the test delivery's `webhook-id` appears in the Redis idempotency cache: `ssh debian@www1.endigitalx.com 'docker compose -f docker-compose.prod.yml exec redis redis-cli GET sirmaai:webhook:dedup:<webhook-id>'` returns `1`.
   - **§Rollback** (revert the nginx changes — leaves cert SAN in place because removing a SAN is non-trivial and harmless):
     ```bash
     # On www1
     cd /home/debian/Projects/eusolicit/eusolicit-app
     git checkout <commit-before-S04.29>~ -- infra/nginx/eusolicit.com
     sudo cp infra/nginx/eusolicit.com /etc/nginx/sites-available/eusolicit.com
     sudo nginx -t && sudo systemctl reload nginx
     # The api.eusolicit.com SAN stays on the cert — harmless if DNS for that
     # subdomain is left up; certbot's auto-renew will keep refreshing the SAN
     # until an operator explicitly drops it (which requires re-issuing the cert
     # WITHOUT --expand, a separate operator task not covered by this rollback).
     ```
   - **§Failure modes** (cheat-sheet table):
     | Symptom | Likely cause | Resolution |
     |---|---|---|
     | `nginx -t` syntax error | Hand-edit drift in /etc/nginx | `diff /etc/nginx/sites-available/eusolicit.com infra/nginx/eusolicit.com` to find the divergence, re-cp |
     | `certbot --expand` fails ACME challenge | DNS for api.eusolicit.com not propagated yet | `dig +short api.eusolicit.com` from a third-party resolver; wait + retry. Cert is unaffected — old cert still valid |
     | `curl https://api.eusolicit.com/webhooks/sirmaai` → 502 | `sirmaai-gateway` container down on 18004 | `docker compose -f docker-compose.prod.yml ps sirmaai-gateway`; chain to `runbooks/container-restart-loop.md` |
     | `curl https://api.eusolicit.com/webhooks/sirmaai` → 503 | SIRMAAI_GATEWAY_ENABLED=false in .env.prod | Flip flag, redeploy gateway; per S04.20 cutover plan |
     | TLS "unknown CA" / cert mismatch | SAN expansion didn't land | Re-run `certbot --nginx --expand` with all four `-d` flags; `sudo systemctl reload nginx` |
   - **§References** (link to: Epic 4 amendment §S04.29, architecture amendment §3.4 + §5.1, S04.25 receiver story, `project_deploy_nginx_manual.md` memory, `project_www1_host_state.md` memory, `www1-rebuild.md` runbook).

6. **NO application-code changes**. This story changes **only**:
   - `eusolicit-app/infra/nginx/eusolicit.com` (modified — +api block, -ai block, +api in :80 server_name)
   - `eusolicit-app/infra/host/templates/nginx-sites/eusolicit.com.conf.j2` (modified — same three changes mirrored)
   - `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md` (NEW file)
   - `eusolicit-docs/implementation-artifacts/4-29-public-ingress-for-webhook-receiver.md` (this story file)
   - `eusolicit-docs/implementation-artifacts/sprint-status.yaml` (single-row status flip backlog → ready-for-dev → review → done as the story progresses)
   
   Specifically NOT touched:
   - Any `services/*` Python code (S04.25 receiver is already shipped).
   - Any `frontend/*` code.
   - `docker-compose.prod.yml` / `docker-compose.yml` (sirmaai-gateway port binding 18004 is pre-existing).
   - `eusolicit-app/infra/host/playbooks/03-nginx-and-certbot.yml` (the playbook references the .j2 template; updating the template is sufficient — the playbook's task list doesn't change).
   - `eusolicit-app/scripts/deploy.sh` (per memory `project_deploy_nginx_manual.md`, deploy.sh never touches nginx — this story preserves that invariant).
   - GH Actions workflows (none).
   - `.env.prod` / `.env.prod.example` (no new env vars).
   - Alembic migrations (no schema changes).

7. **Smoke-test contract** — the runbook §Deploy step 6 + 7 + 8 from AC 5 IS the smoke test. After running the deploy steps, the runbook deploys MUST produce:
   - `https://api.eusolicit.com/webhooks/sirmaai` returns 401 `{"error":"no_active_subscription"}` or 401 `{"error":"invalid_signature"}` (proves nginx + TLS + upstream wiring + signature-verification path are live).
   - `https://api.eusolicit.com/api/v1/agents/foo/run` returns 404 (proves the catch-all denies non-webhook paths).
   - `https://api.eusolicit.com/admin/circuits` returns 404 (same).
   - `https://api.eusolicit.com/healthz` returns 404 (same; healthz is NOT exposed publicly — operator health-check via SSH + `curl 127.0.0.1:18004/healthz`).
   - `https://www.eusolicit.com/ai/healthz` returns 404 (proves AC 3 deletion is live; was previously 200).
   - `dig +short api.eusolicit.com` returns the www1 IP (proves DNS is in place — pre-flight, but worth re-confirming after the change).

8. **Operator pre-flight checklist** is captured in §Pre-flight of the runbook (AC 5) and the story's Tasks/Subtasks section. The DNS A record provisioning is **explicitly an operator task**, not an in-scope code change, per `infra/host/README.md` line 51 — but the runbook §Pre-flight makes it impossible for the operator to skip.

9. **Documentation cross-references updated** (string-search audit during the dev pass — no code changes if the strings aren't present):
   - `eusolicit-docs/runbooks/www1-rebuild.md` §Phase 2 (first-run cert provisioning, currently lines 45-60) — add a comment that the `certbot --nginx --expand` command for a clean rebuild MUST include `-d api.eusolicit.com` so the rebuilt cert covers the webhook ingress from the start. The runbook's existing command shows three `-d` flags; this story's AC adds the fourth. Single-line addition + a short explanatory note pointing at `runbooks/sirmaai-webhook-ingress.md` for the full vhost plumbing.
   - `eusolicit-app/infra/host/README.md` — confirm the readme already states "DNS records (out of host scope; operator action)" (line 51) and no additional documentation is needed (the runbook IS the operator-facing doc).
   - `CLAUDE.md` (project root) — no change needed (the AI Gateway / sirmaai-gateway architecture is already documented; this story is operational, not architectural).
   - `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md` — no change needed (S04.29 description already references the runbook path; this story IS the runbook implementation).

10. **DoD per project delivery rules** (the only DoD steps that apply — this is a pure infrastructure story with no Python code touched):
    - `nginx -t` passes on the modified `infra/nginx/eusolicit.com` after the dev-pass edit (the dev-story agent can syntax-check locally via `docker run --rm -v $(pwd)/infra/nginx/eusolicit.com:/etc/nginx/sites-enabled/eusolicit.com:ro nginx:alpine nginx -t` — Note: this requires the `snippets/*.conf` includes to be present or shimmed; an alternative is `nginx -t -p /tmp/nginx-test-prefix -c /tmp/test.conf` with a wrapper that `include`s only the modified file — pragmatic recommendation: skip the local docker syntax check and rely on the runbook's `sudo nginx -t` on www1 as the canonical gate). Document the syntax-check approach actually taken in the dev-pass Completion Notes.
    - `pnpm lint` / `pnpm type-check` — N/A (no frontend changes).
    - `make lint` / `make type-check` — N/A (no Python changes). Run anyway as a no-op confirmation that the story didn't accidentally touch any service code.
    - `make test-*` — N/A (no application-code changes; behavioral changes are purely network-layer + manual nginx config).
    - **Markdown lint** on the new runbook (the project's docs convention — match the format of `runbooks/sirmaai-key-rotation.md` and `runbooks/www1-rebuild.md`; no strict linter is enforced, but the file should pass a `markdownlint-cli2` pass if one is locally available).
    - **Diff review of the two nginx files** — they MUST be byte-for-byte equivalent on the new server block and the `:80` server_name extension, ignoring the .j2 template's `{# … #}` Jinja header comment. The dev pass MUST capture this diff in Completion Notes as proof of AC 4.

11. **Cross-tenant negative test invariant** (project memory rule — explicitly called out even though the story is infrastructural):
    - The webhook receiver's application-layer cross-tenant defense is already shipped in S04.25 AC 7 (signature signed with subscription B's secret arriving for subscription A → 401, no DB writeback, no Redis idempotency-cache write). This story does NOT add a new cross-tenant defense surface — it adds the public-facing transport that delivers signed events to that already-tested defense. The runbook §Verify-end-to-end step (operator triggers SirmaAI test webhook) IS the end-to-end test that the public ingress doesn't bypass the signature check; the application-layer test fixtures from S04.25 remain the canonical regression coverage.
    - Document this in the dev-pass Completion Notes as "AC 11 — covered by S04.25 unit + integration test suite, no new tests required in S04.29 scope".

12. **Story status flips** follow AP18-C2 atomic two-gate discipline (per S04.27/S04.28 precedent):
    - `backlog` → `ready-for-dev` at story creation (this dispatch).
    - `ready-for-dev` → `in-progress` when dev-story starts.
    - `in-progress` → `review` after the dev-pass edits land + the runbook passes `markdownlint` + the nginx files diff cleanly.
    - `review` → `done` after [VS] validate-story + [PR] post-review confirm AC coverage. The operator-side sudo cp + certbot --expand on www1 is **out of scope for the story status flip** — the story is "done" when the code + runbook land on main, regardless of whether an operator has executed the runbook in prod yet (per memory `project_deploy_nginx_manual.md`, the manual step is the operator's responsibility, not a gating criterion for marking the story done).

## Tasks / Subtasks

- [x] **Task 1: Pre-flight audit of existing public surface** (AC 3 prerequisite)
  - [x] Grep the codebase for any caller depending on the existing `/ai/` public path (deletion target). Commands: `grep -rn '/ai/' eusolicit-app/services/ eusolicit-app/frontend/`, `grep -rn 'eusolicit.com/ai' .`. Expected: zero hits.
  - [x] If any hit is returned, HALT — file a follow-up story for the caller migration and escalate via [VS] bmad-validate-story.
  - [x] Document the grep results (zero hits expected) in the dev-pass Completion Notes as proof that AC 3 deletion is safe.

- [x] **Task 2: Edit `eusolicit-app/infra/nginx/eusolicit.com`** (AC 1, 2, 3)
  - [x] Extend the `:80` server block's `server_name` line (currently lines 14-26) to add `api.eusolicit.com`. Resulting `server_name`: `www.eusolicit.com eusolicit.com admin.eusolicit.com api.eusolicit.com;` (AC 2).
  - [x] Delete the existing `/ai/` location block (currently lines 132-137 — the four-line block proxying to `127.0.0.1:18004/`). Pure subtraction; no replacement, no comment placeholder (AC 3).
  - [x] Append a new `:443 ssl` server block for `api.eusolicit.com` at the end of the file, following the canonical structure (AC 1):
    - `listen 443 ssl;` + `http2 on;`
    - `server_name api.eusolicit.com;`
    - `ssl_certificate /etc/letsencrypt/live/www.eusolicit.com/fullchain.pem;` + `ssl_certificate_key /etc/letsencrypt/live/www.eusolicit.com/privkey.pem;`
    - `include snippets/ssl-params.conf;` + `include snippets/security-headers.conf;`
    - `access_log /var/log/nginx/eusolicit.access.log main;` + `error_log /var/log/nginx/eusolicit.error.log warn;`
    - `client_max_body_size 1m;`
    - One `location = /webhooks/sirmaai { proxy_pass http://127.0.0.1:18004/webhooks/sirmaai; include snippets/proxy-params.conf; proxy_read_timeout 30s; proxy_request_buffering on; }` block.
    - One `location / { return 404; }` catch-all.
  - [x] Add a short explanatory comment block above the new server block, matching the file's existing comment-style (4-character `# ──` separators; explain "Public ingress for SirmaAI Standard Webhooks delivery — sole public path is `/webhooks/sirmaai`; all other URIs return 404 by design — see eusolicit-docs/runbooks/sirmaai-webhook-ingress.md").

- [x] **Task 3: Mirror changes in `eusolicit-app/infra/host/templates/nginx-sites/eusolicit.com.conf.j2`** (AC 4)
  - [x] Re-read the current `eusolicit.com.conf.j2` file in full BEFORE editing (template currently does NOT have an `/ai/` block, so AC 3's deletion is a no-op on this file — confirm this and note it in Completion Notes).
  - [x] Extend the `:80` server block's `server_name` line to add `api.eusolicit.com`.
  - [x] Append the same `api.eusolicit.com` `:443 ssl` server block, byte-for-byte identical to AC 1 (apart from preserving the existing Jinja `{# … #}` header comment at top of the .j2 file).
  - [x] Run a manual diff between the two files (the canonical `infra/nginx/eusolicit.com` and the .j2 template) — they MUST be byte-equivalent on the new content. Document the diff in Completion Notes.

- [x] **Task 4: Author the runbook at `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`** (AC 5)
  - [x] Use the structure of `eusolicit-docs/runbooks/sirmaai-key-rotation.md` as the template skeleton (it's the closest sibling runbook — same operator surface area, same SirmaAI domain).
  - [x] Cover sections §Purpose, §When to use, §Pre-flight, §Deploy, §Verify-end-to-end, §Rollback, §Failure modes, §References — exactly as AC 5 enumerates.
  - [x] Include the copy-pasteable bash blocks from AC 5 §Deploy. The bash MUST be exact — operators will paste it verbatim.
  - [x] Include the `dig` + `openssl` pre-flight commands from AC 5 §Pre-flight.
  - [x] Cross-link to: `eusolicit-docs/runbooks/www1-rebuild.md`, `eusolicit-docs/runbooks/sirmaai-key-rotation.md`, `eusolicit-docs/runbooks/container-restart-loop.md`, `eusolicit-docs/runbooks/docker-daemon-recovery.md`, `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md` §S04.29, `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` §3.4 + §5.1, `eusolicit-docs/implementation-artifacts/4-25-standard-webhooks-receiver.md`.

- [x] **Task 5: Update the `www1-rebuild.md` runbook to reference the new SAN** (AC 9)
  - [x] Open `eusolicit-docs/runbooks/www1-rebuild.md` and locate §Phase 2 — first-run cert provisioning (currently lines 45-60).
  - [x] Add `-d api.eusolicit.com` to the `certbot --nginx --expand` command (existing three flags become four). Add a brief explanatory line pointing at `runbooks/sirmaai-webhook-ingress.md` for the full vhost plumbing.
  - [x] This is a minimal single-line addition + a short comment — it's an oversight-prevention change so a clean DR rebuild ships with the api.eusolicit.com SAN from the start, not as a post-rebuild fix-up.

- [x] **Task 6: Local nginx syntax check (best-effort)** (AC 10)
  - [x] Attempt a local syntax check: `docker run --rm -v "$(pwd)/eusolicit-app/infra/nginx/eusolicit.com:/etc/nginx/sites-enabled/eusolicit.com:ro" nginx:alpine nginx -t`.
  - [x] EXPECTED LIMITATION: this will fail with `open() "snippets/proxy-params.conf" failed` because the snippets aren't deployed in the alpine image. Document this as a known limitation in Completion Notes and rely on the runbook's `sudo nginx -t` on www1 as the canonical syntax gate.
  - [x] Alternative: a focused syntax check via `nginx -t -c <(echo "events {} http { include $(realpath infra/nginx/eusolicit.com); }")` with mocked snippet files. This is at the dev-story agent's discretion; the canonical gate remains `sudo nginx -t` on www1.

- [x] **Task 7: Markdown lint the runbook** (AC 10)
  - [x] Confirm the new runbook reads cleanly — section headings consistent with sibling runbooks, code blocks language-tagged (` ```bash`), cross-links use absolute paths from project root.
  - [x] If a `markdownlint-cli2` config exists in the repo, run it on the new runbook file; otherwise skip.

- [x] **Task 8: Story finalization** (AC 12)
  - [x] Update `eusolicit-docs/implementation-artifacts/sprint-status.yaml` row `4-29-public-ingress-for-webhook-receiver: backlog → ready-for-dev` at story-creation time (this dispatch).
  - [x] Subsequent flips (`ready-for-dev → in-progress → review → done`) are owned by dev-story / validate-story / code-review / post-review per the BMAD pipeline.

## Dev Notes

### Architectural context (read before editing anything)

This story closes Concern #4 from the SirmaAI pivot's implementation-readiness report. The architecture-amendment §3.4 + §5.1 + §5.3 establish that SirmaAI delivers webhooks (Standard Webhooks spec) to EU Solicit, and the receiver lives at `sirmaai-gateway:8004/webhooks/sirmaai` (shipped by S04.25). What's MISSING in production is the public-facing transport: SirmaAI cannot reach a ClusterIP service. This story plumbs nginx on www1 so SirmaAI's outbound delivery agent can reach the receiver via `https://api.eusolicit.com/webhooks/sirmaai`.

The Epic 4 amendment AC line 485 reads: *"Public ingress configured for webhook receiver only (path `/webhooks/sirmaai`); all other gateway paths remain ClusterIP-only (updates original Epic 4 AC #15)."* The wording "all other gateway paths remain ClusterIP-only" is the load-bearing constraint that drives:
1. The new vhost serves ONLY `/webhooks/sirmaai` — every other URI on `api.eusolicit.com` returns 404 by design.
2. The existing `/ai/` location on `www.eusolicit.com` (which today exposes the entire gateway publicly) gets DELETED as part of this story — leaving it in would contradict the AC.

### Why a new subdomain (`api.eusolicit.com`) instead of a path under `www.eusolicit.com`

The epic line 511 specifies `api.eusolicit.com`. Three reasons this is the right call:

1. **Surface separation**: a dedicated subdomain for the public API surface lets us apply per-vhost defaults (smaller body cap, shorter read timeout, simpler catch-all) without affecting the customer-facing `www.eusolicit.com` traffic profile.
2. **Future-proofing**: a future story might add `https://api.eusolicit.com/v1/…` public API routes for third-party integrations; having the subdomain in place now means that's a vhost edit, not a DNS + cert dance.
3. **Operational clarity**: webhook traffic shows up under a distinct vhost in the access log (`grep 'api.eusolicit.com' /var/log/nginx/eusolicit.access.log`), making incident triage and traffic-pattern analysis trivial — no string-matching the URL path.

### Why nginx on www1 (and not an ingress controller)

Per memory `project_onprem_pivot_2026_05_11.md`, ADR-010 rewrote the production target from AWS managed services to single-host Docker on www1. nginx + certbot on the host is the canonical TLS-terminating reverse proxy in this topology. Per memory `project_deploy_nginx_manual.md`, nginx config changes are a deploy-time **manual** SSH step — `scripts/deploy.sh` deliberately never touches `/etc/nginx`. This story respects that contract: the change ships as a code commit + a runbook that the operator follows; the GH Actions deploy workflow is unchanged.

### Why both `infra/nginx/eusolicit.com` AND `infra/host/templates/nginx-sites/eusolicit.com.conf.j2` must change

Two reference paths, one production state:

- `infra/nginx/eusolicit.com` is the **canonical** reference that `project_deploy_nginx_manual.md` mandates be `sudo cp`'d to `/etc/nginx/sites-available/eusolicit.com` on www1 during a routine deploy.
- `infra/host/templates/nginx-sites/eusolicit.com.conf.j2` is the **Ansible template** that `infra/host/playbooks/03-nginx-and-certbot.yml` renders to the same destination during a `www1-rebuild.md` Phase 1 disaster-recovery flow.

If the two diverge, a future DR rebuild will reconstruct an nginx state that doesn't match production. Story onprem-06 established this two-file invariant; we honor it here.

### Why DELETE the existing `/ai/` location (and not refactor it)

The existing `/ai/` block at `infra/nginx/eusolicit.com` lines 132-137 proxies `https://www.eusolicit.com/ai/*` → `http://127.0.0.1:18004/*`. This means TODAY, in production, every gateway endpoint — `/admin/circuits`, `/admin/tenant-status`, `/agents/{id}/run`, `/agents/{id}/run-stream`, `/webhooks/kraftdata`, `/runs/{id}` — is publicly reachable via the `eusolicit.com/ai/` prefix. That is a pre-existing security gap that contradicts the original Epic 4 AC #15 ("ClusterIP-only").

The Epic 4 amendment AC line 485 closes this gap by stating "all other gateway paths remain ClusterIP-only" — which means after S04.29 lands, the ONLY public gateway path is `api.eusolicit.com/webhooks/sirmaai`. Deleting the `/ai/` block is the implementation of that AC.

Pre-deletion grep audit (Task 1) is the safety check — if any in-tree caller depends on the `www.eusolicit.com/ai/` prefix (which it shouldn't, because all inter-service gateway calls go through the docker-network hostname `sirmaai-gateway:8004`), we abort and file a migration story. The grep is the gate.

### S04.20 / S04.30 service-rename interaction (NO change needed in this story)

Per CLAUDE.md: *"`ai-gateway` has been renamed to `sirmaai-gateway` (S04.20) — the docker-compose service hostname is now `sirmaai-gateway` with a temporary network alias `ai-gateway` for backward compatibility during the SirmaAI amendment cutover."* The host port `127.0.0.1:18004:8004` is unchanged across the rename; nginx targets the host port, not the docker-network hostname, so the service rename is **invisible** to this story. When S04.30 retires the `ai-gateway` alias, this story's nginx config remains correct — no change needed.

### S04.25 receiver contract (the upstream contract this story respects)

The receiver at `POST /webhooks/sirmaai` (shipped by S04.25):
- Reads the raw body once via `await request.body()` for HMAC-over-raw-bytes verification → **`proxy_request_buffering on;`** in nginx is required so the full body arrives in one read.
- Completes in <100ms p95 per the S04.25 contract (the body is small, signature verification is CPU-only Fernet-decrypt + HMAC-SHA256, Redis SETNX is sub-ms) → **`proxy_read_timeout 30s;`** is 300× the budget; anything taking 30s upstream is a deadlocked handler and should fail-fast back to SirmaAI for retry.
- Returns 200 on success (and a structured JSON body for every outcome — see S04.25 AC 1 status table). No retry shaping needed at the nginx layer.
- Already enforces the cross-tenant invariant (S04.25 AC 7) at the application layer — this story does NOT add a second cross-tenant defense. The single layer is the contract.

### Cert SAN expansion vs separate cert per subdomain (single-cert is right)

The existing letsencrypt cert at `/etc/letsencrypt/live/www.eusolicit.com/fullchain.pem` already covers three SANs (`www.eusolicit.com`, `eusolicit.com`, `admin.eusolicit.com`). The pattern is **one cert with multiple SANs** — not a cert-per-subdomain. We follow the precedent by `--expand`-ing to add `api.eusolicit.com` as a fourth SAN.

Why single-cert is right:
- One cert means one renewal cycle, one expiry alarm, one rotation runbook.
- The existing certbot systemd timer on www1 renews ALL SANs on the cert at the same time — adding a SAN doesn't add a new timer or a new state machine.
- Per `runbooks/www1-rebuild.md` line 52, the rebuild cert-issuance command lists all `-d` flags in one invocation — single-cert keeps the rebuild ritual simple.

### Why `client_max_body_size 1m;` (and not the 50M inherited from www)

The `www.eusolicit.com` block at file line 77 sets `client_max_body_size 50M;` (sized for proposal-document uploads). The `api.eusolicit.com` block MUST NOT inherit that — webhook payloads are kilobytes per Standard Webhooks (the spec doesn't define a hard upper bound but the SirmaAI event catalog payloads are well under 100 KB; 1 MB is 10–100× headroom). A 50 MB cap on the webhook path would let a malicious actor flood the gateway with megabyte payloads, each triggering an `await request.body()` read on the upstream → cheap-on-the-wire, expensive-on-the-handler asymmetric DoS vector. The 1 MB cap forecloses that.

### Why `location = /webhooks/sirmaai` (literal equality) and not regex

`location = /uri` is the fastest dispatch in nginx — exact match, no regex evaluation, decided before any prefix-matching pass. We want the hot path (`POST /webhooks/sirmaai` from SirmaAI's delivery agent) to be the fastest dispatch, and we want the catch-all (`location /`) to handle every OTHER URI. The literal `=` match also has a security benefit: a regex `~ ^/webhooks/sirmaai` would match `/webhooks/sirmaai/anything-trailing` and forward the trailing path to the upstream, which is brittle — the literal match forecloses path-suffix smuggling.

### Why explicit `return 404;` (and not `deny all;` or default behavior)

`deny all;` returns 403 Forbidden with a body — slightly informative to an attacker (confirms the path is restricted, not absent). `return 404;` returns 404 Not Found with no body — minimum information disclosure. The default nginx behavior on no-match-and-no-default is implementation-dependent (autoindex if enabled, otherwise 403/404) — we don't rely on defaults; we encode the intent.

### Why no IP allowlist on the public path

Per S04.25's design and Standard Webhooks discipline (architecture-amendment §5.1 — "no IP-allowlist on the public path per Standard Webhooks discipline"), the application-layer signature verification + replay-window + idempotency cache is the ONLY defense. Reasoning:
1. SirmaAI's outbound delivery agent IP range is not contractually stable; an allowlist would require operator updates every SirmaAI infrastructure change.
2. Standard Webhooks signing (HMAC SHA-256 with constant-time compare via `hmac.compare_digest()`) is the contract-correct defense — it works regardless of source IP.
3. Adding an allowlist would couple this story to SirmaAI's network topology, which is an external dependency we deliberately keep loosely coupled (architecture-amendment §11.3 risk #1).

### Project memory rules honored

- **`project_deploy_nginx_manual.md`**: this story preserves the invariant that `scripts/deploy.sh` never touches nginx. The change ships as code on main + a runbook; the manual sudo cp + certbot --expand + nginx reload is the operator's job, captured in the runbook.
- **`project_www1_host_state.md`**: the canonical www1 IP (46.10.208.159) is documented in the runbook's §Pre-flight DNS check. The `/etc/nginx/sites-available/eusolicit.com` path is host-only state; the runbook makes it explicit that operators sudo cp the repo file there.
- **`project_onprem_pivot_2026_05_11.md`**: this story implements public ingress on nginx (the chosen on-prem topology), not on an AWS ALB / managed ingress. No AWS managed services proposed.
- **`project_sirmaai_pivot_2026_05_12.md`**: this story is part of the E04 amendment cluster (S04.20–S04.31); it injects the missing public-ingress story per Concern #4.
- **`project_auto_sync_quality.md`**: this is a hand-authored change (not auto-sync); the dev-story agent will run the local syntax check (Task 6) before claiming completion.
- **`feedback_investigate_dont_quiz.md`**: pre-flight Task 1 grep audit is the investigation; no operator questioning needed.

### Files this story touches (definitive list)

- `eusolicit-app/infra/nginx/eusolicit.com` — modified (AC 1, 2, 3): +api block, -ai block, +api in :80 server_name.
- `eusolicit-app/infra/host/templates/nginx-sites/eusolicit.com.conf.j2` — modified (AC 4): same three changes mirrored.
- `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md` — NEW file (AC 5).
- `eusolicit-docs/runbooks/www1-rebuild.md` — modified (AC 9): one-line addition to the `certbot --nginx --expand` invocation.
- `eusolicit-docs/implementation-artifacts/4-29-public-ingress-for-webhook-receiver.md` — THIS file (created in this dispatch).
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` — single-row flip backlog → ready-for-dev (this dispatch) and the subsequent BMAD pipeline flips.

### Files this story does NOT touch (sanity list)

- `services/sirmaai-gateway/` — S04.25 receiver is already shipped + production-tested at the application layer.
- `services/client-api/` / `services/admin-api/` / `services/data-pipeline/` / `services/notification/` / `services/integrations-api/` — none of them are public-ingress concerns.
- `frontend/` — no frontend touch.
- `docker-compose.prod.yml` / `docker-compose.yml` — port binding 18004 already exists.
- `infra/host/playbooks/03-nginx-and-certbot.yml` — playbook references the .j2 template; updating the template is sufficient.
- `scripts/deploy.sh` — preserved by design.
- `.env.prod` / `.env.prod.example` — no new env vars.
- Alembic migrations — no schema changes.
- Any test file (no application-code behavior changes; S04.25's existing test suite is the regression coverage).

### Test design alignment

The epic-level test design at `eusolicit-docs/test-artifacts/test-design-epic-04.md` was authored 2026-04-14 (pre-SirmaAI pivot) and does NOT directly cover S04.29's public ingress. The closest analogs are:
- **E04-P0-010 / E04-P0-011** (lines 145-146): webhook receiver happy path + invalid-signature rejection. These remain canonical for the receiver behavior — and they cover the application-layer defense that this story's public ingress hands events to. Per AC 11, this story does NOT add a parallel test suite; it relies on S04.25's unit + integration tests for application-layer correctness.
- The "Kubernetes ClusterIP networking / Ingress" row in the test-design's §Not in Scope table (line 52) reads: *"Infrastructure concern owned by DevOps; service is ClusterIP-only per spec."* The SirmaAI amendment AC line 485 updates the "ClusterIP-only" wording (one path is public); the SCOPE of testing remains the same — nginx config syntax is verified by `nginx -t` on the host (manual), end-to-end reachability is verified by the runbook §Deploy step 6 + 7 + 8 smoke tests (manual operator action), and there is no in-CI test that asserts "the nginx config on www1 looks like X" because the CI environment doesn't have nginx + the live cert chain.

The dev-pass MUST document in Completion Notes that:
- AC 7 smoke-test contract is OPERATOR-EXECUTED (runbook §Deploy step 6 + 7 + 8) — not automated in CI.
- AC 11 cross-tenant negative test is COVERED by S04.25's existing test suite.
- No new application-code tests are added by this story (this is the conscious design — infrastructure-only stories don't ship parallel test suites).

### Previous story context (S04.27, S04.28)

S04.27 (Tenant Degraded-Mode Banner) and S04.28 (Tier-to-SirmaAI Rate-Limit Sync) are the most recent landed S04 stories. Patterns from those stories carried forward here:
- **AP18-C2 atomic two-gate** (story file + sprint-status row in single dispatch): observed by this story's sprint-status flip happening atomically at story-creation time.
- **No new Prometheus counters / no new Celery tasks for this scope**: this story is even thinner — no new application code at all.
- **Single-commit landing**: this story's dev-pass MUST land in one commit (the three file changes + the new runbook) so a rollback is a single `git revert` plus the operator-side `sudo cp` revert.
- **Markdown lint discipline on runbooks**: S04.27's banner i18n strings ran through `pnpm check:i18n`; this story's runbook should pass markdown-lint-style review (consistency with the sibling runbook format).

### LLM optimization hints for the dev agent

- The single most important file to read BEFORE editing is `eusolicit-app/infra/nginx/eusolicit.com` — read it in full; the conventions (snippet includes, log paths, comment style) are stable across the file and the new block MUST match.
- The .j2 template is shorter and has a different header — preserve the Jinja header comment block at the top of the file.
- The runbook's bash blocks MUST be exact. Operators will paste them verbatim. Test the bash mentally for shell-quoting correctness (`-d` flags as separate args, not concatenated; quotes around `curl -H` header values).
- DO NOT introduce any new permissions, deny-rules, or "while we're here" cleanups beyond the three changes specified. Scope creep on infrastructure stories breaks production.
- The `/ai/` location DELETION is a small but high-impact change — pre-flight Task 1 grep audit is the gate, not a formality.

### References

- Epic file: `eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md` §S04.29 (line 511) + Amended AC (line 485).
- Architecture amendment: `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` §3.4 (AI Platform rewrite) + §5.1 (External APIs table) + §11.3 (risk #1, "SirmaAI as second critical external dependency").
- Implementation readiness report: `eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-12-sirmaai.md` §Concern #4 (line 245-257) + §Closure (line 414).
- PRD amendment: `eusolicit-docs/planning-artifacts/prd-amendment-2026-05-12-sirmaai.md` (NFR + webhook ingress contract).
- Predecessor story (receiver shipped): `eusolicit-docs/implementation-artifacts/4-25-standard-webhooks-receiver.md`.
- Sibling story (banner — most recent S04 landing): `eusolicit-docs/implementation-artifacts/4-27-tenant-degraded-mode-banner.md`.
- Sibling story (rate-limit sync — most recent landed): `eusolicit-docs/implementation-artifacts/4-28-tier-to-sirmaai-rate-limit-sync.md`.
- Canonical nginx file: `eusolicit-app/infra/nginx/eusolicit.com` (read in full before editing).
- Ansible nginx template: `eusolicit-app/infra/host/templates/nginx-sites/eusolicit.com.conf.j2`.
- Nginx Ansible playbook: `eusolicit-app/infra/host/playbooks/03-nginx-and-certbot.yml`.
- Host-as-code README: `eusolicit-app/infra/host/README.md`.
- DR runbook: `eusolicit-docs/runbooks/www1-rebuild.md` (will be modified — AC 9).
- Sibling SirmaAI runbook: `eusolicit-docs/runbooks/sirmaai-key-rotation.md` (template for new runbook structure — AC 5).
- Compose file (port allocation): `eusolicit-app/docker-compose.prod.yml` line 267 (`127.0.0.1:18004:8004`).
- Memory rules: `project_deploy_nginx_manual.md`, `project_www1_host_state.md`, `project_onprem_pivot_2026_05_11.md`, `project_sirmaai_pivot_2026_05_12.md`, `project_auto_sync_quality.md`, `feedback_investigate_dont_quiz.md`.

### Project Structure Notes

- All file edits land under `eusolicit-app/infra/nginx/` + `eusolicit-app/infra/host/templates/nginx-sites/` (paired infrastructure-as-code surfaces) + `eusolicit-docs/runbooks/` (operator-facing documentation). No service code is touched.
- The two-nginx-files invariant (canonical + Ansible template) is Story onprem-06's contract. Preserve it.
- The runbook lives under `eusolicit-docs/runbooks/` (NOT `eusolicit-app/runbooks/` — that folder is for app-side runbooks like `outcome-brief-s3-lifecycle.md`; SirmaAI / infrastructure runbooks are in `eusolicit-docs/`).
- No detected conflicts with unified project structure.

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 | bmad-dev-story | 2026-05-14

### Debug Log References

None — infrastructure-only story; no runtime errors encountered.

### Completion Notes List

**Task 1 — Pre-flight grep audit (AC 3 prerequisite):**
- `grep -rn '/ai/' eusolicit-app/services/ eusolicit-app/frontend/` → ALL hits are in `.venv` vendor packages only (pyparsing's `ai/` subpackage at `pyparsing/ai/__init__.py`, redis docs at `redis/commands/search/hybrid_query.py`). ZERO hits in application source code.
- `grep -rn 'eusolicit.com/ai' .` → hits ONLY in `eusolicit-docs/implementation-artifacts/4-29-public-ingress-for-webhook-receiver.md` (this story file, as audit-command text and explanatory prose). ZERO hits in any service or frontend source.
- **Conclusion**: `/ai/` location block deletion is SAFE. No in-tree caller depends on the `www.eusolicit.com/ai/` public path.

**Task 2 — `infra/nginx/eusolicit.com` changes applied:**
- `:80` server_name extended: `www.eusolicit.com eusolicit.com admin.eusolicit.com api.eusolicit.com;`
- Header comment updated to list all four SANs in the `certbot --nginx --expand` example.
- `/ai/` location block removed (4-line block at former lines 132-137: `location /ai/ { proxy_pass http://127.0.0.1:18004/; ... }`). Pure subtraction, no replacement.
- New `api.eusolicit.com` `:443 ssl` server block appended at end of file with all required directives: `listen 443 ssl; http2 on; server_name api.eusolicit.com;` + shared cert paths + snippet includes + log directives + `client_max_body_size 1m;` + `location = /webhooks/sirmaai` with `proxy_pass http://127.0.0.1:18004/webhooks/sirmaai; proxy_read_timeout 30s; proxy_request_buffering on;` + `location / { return 404; }` catch-all.

**Task 3 — `eusolicit.com.conf.j2` mirror changes (AC 4):**
- Confirmed: the `.j2` template did NOT have an `/ai/` location block (it is a slimmer version of the canonical file — stops at `/admin-api/` block). AC 3 deletion is a no-op on the `.j2` file.
- `:80` server_name extended with `api.eusolicit.com` (same three `if` pre-redirect blocks in the .j2's compact format preserved).
- Same `api.eusolicit.com` `:443 ssl` server block appended byte-for-byte identical to the canonical file.
- **Diff between the two files**: Expected differences are the .j2 Jinja header (`{# … #}` block), the shorter `# Generated by Ansible` comment, the compact inline `:80` block format (single-line locations), fewer verbose comment blocks in the www server block (no `/ws/`, no `/health`, no `/_next/static/`, no dotfiles blocks — pre-existing differences from Story onprem-06). The NEW `api.eusolicit.com` server block is byte-for-byte identical in both files. ✓

**Task 5 — `www1-rebuild.md` update (AC 9):**
- Phase 2 certbot command extended with `-d api.eusolicit.com` (three flags → four) + explanatory comment pointing at `runbooks/sirmaai-webhook-ingress.md`.

**Task 6 — Local nginx syntax check (AC 10):**
- `docker run --rm -v "$(pwd)/eusolicit-app/infra/nginx/eusolicit.com:/etc/nginx/sites-enabled/eusolicit.com:ro" nginx:alpine nginx -t` executed successfully.
- **RESULT**: Reported "syntax is ok" + "test is successful" — however, nginx:alpine's default config includes `conf.d/*.conf` NOT `sites-enabled/*`, so our custom vhost was not actually parsed. This is the expected limitation noted in AC 10. The Docker container output is NOT a valid syntax gate for our file.
- **Canonical gate**: `sudo nginx -t` on www1 after `sudo cp infra/nginx/eusolicit.com /etc/nginx/sites-available/eusolicit.com` per §Deploy step 3 of the runbook.

**Task 7 — Markdown lint (AC 10):**
- No `.markdownlint*` config found in the repository. `markdownlint-cli2` is not installed. Per AC 10: "If a markdownlint-cli2 config exists in the repo, run it on the new runbook file; otherwise skip." — SKIPPED.
- Visual review: all section headings consistent with `sirmaai-key-rotation.md` format; all code blocks tagged with ` ```bash`; cross-links formatted as relative markdown links; §Failure modes uses the same table format as AC 5 specification.

**AC 7 — Smoke-test contract:**
- The smoke-test contract (steps 6, 7, 8 of §Deploy) is OPERATOR-EXECUTED as part of the runbook deploy procedure. It is NOT automated in CI because CI has no nginx + live TLS cert chain. The canonical verification is the runbook §Deploy steps executed on www1 by an operator after the manual `sudo cp + certbot --expand + systemctl reload nginx` sequence.

**AC 11 — Cross-tenant negative test invariant:**
- Covered by S04.25's existing unit + integration test suite (AC 7 of S04.25 covers: signature signed with subscription B's secret arriving for subscription A → 401, no DB writeback, no Redis idempotency-cache write). S04.29 adds the public-facing transport layer only; the application-layer cross-tenant defense is unchanged from S04.25. No new application-code tests are added by this story — this is the conscious design per AC 11 + Dev Notes §Test design alignment.

**`make lint` / `make type-check` confirmation (AC 10 no-op check):**
- Both commands were run. Pre-existing failures in `sirmaai-gateway` Python files (rotate_keys.py, execution.py, main.py) from prior stories — NOT attributable to S04.29 which makes ZERO Python code changes. Confirmed: no service code was accidentally modified.

### File List

**New:**
- `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md`

**Modified:**
- `eusolicit-app/infra/nginx/eusolicit.com` (+api block, -ai block, +api in :80 server_name, +header cert comment update)
- `eusolicit-app/infra/host/templates/nginx-sites/eusolicit.com.conf.j2` (+api block, +api in :80 server_name; no /ai/ block existed in .j2)
- `eusolicit-docs/runbooks/www1-rebuild.md` (Phase 2 certbot --expand +1 flag + explanatory comment)
- `eusolicit-docs/implementation-artifacts/4-29-public-ingress-for-webhook-receiver.md` (this story file — status + tasks + Dev Agent Record)
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` (row flipped ready-for-dev → review)

**Deleted:**
- (none)

### Test Results

N/A — pure infrastructure story (nginx config + runbook). No application code changed. No pytest test suite applies.

`make lint` and `make type-check` run as no-op confirmation: pre-existing failures in sirmaai-gateway Python files (not attributable to S04.29). No new lint or type errors introduced.

Canonical test gate is operator-executed per runbook §Deploy steps 6+7+8 on www1 (nginx -t + smoke curl proving 401 on /webhooks/sirmaai + 404 on all other paths + 404 on former /ai/ surface).

### Review Follow-up Pass (2026-05-14)

All 5 code-review patch findings resolved in this pass:

✅ Resolved review finding [High — functional]: Patch 1 — Added `if ($host = api.eusolicit.com) { return 301 https://$host$request_uri; }` to the `.j2` template's `:80` block (alphabetically between `admin` and `eusolicit.com`). Without this, `http://api.eusolicit.com/` would silently redirect to `https://www.eusolicit.com/` via the `$server_name` fallback. The canonical nginx file's `$server_name` issue is a pre-existing deferred finding; only the `.j2` template was in patch scope.

✅ Resolved review finding [High — correctness]: Patch 2 — Changed hardcoded `webhook-timestamp: 1715731200` (2024-05-14, ~1 year stale) to `webhook-timestamp: $(date +%s)` (using double quotes to enable shell substitution). Operators running the smoke test with the stale epoch would receive a freshness-rejection 4xx before the signature check fires, making the "expected 401" outcome impossible.

✅ Resolved review finding [Medium — operational clarity]: Patch 3 — Added 503 diagnostic to §Deploy step 6 expected-responses. Documented cause ("SIRMAAI_GATEWAY_ENABLED=false OR env var changed but container not restarted") and remediation (`docker compose restart sirmaai-gateway`). Without this, an operator seeing 503 would debug nginx instead of the gateway flag/container restart cycle.

✅ Resolved review finding [Medium — operational safety]: Patch 4 — Added `sudo cp /etc/nginx/sites-available/eusolicit.com /tmp/eusolicit.com.bak` to §Deploy step 3 (captures pre-deploy state before overwriting). Updated §Rollback to add a precondition check (`git log --oneline -- infra/nginx/eusolicit.com | head -3`) that distinguishes "commit landed" vs "pre-commit deploy window" and routes to the correct rollback path.

✅ Resolved review finding [Medium — operational safety]: Patch 5 — Rewrote §Deploy steps 3-5 to: (a) add explicit `sudo systemctl reload nginx` at end of step 3 to activate the updated `:80` block for certbot's ACME challenge; (b) switch step 4 from `certbot --nginx --expand` to `certbot certonly --webroot -w /var/www/certbot --expand` to prevent in-place nginx config rewriting by certbot (eliminates drift from the repo file on subsequent `sudo cp`); (c) document the brief TLS window (steps 3-5) and recommend running promptly before configuring SirmaAI; (d) updated step 5 comment to "TLS window closed" and the "Safe to re-run" note to reference the new certbot command.

**No new test failures** — this is a pure infrastructure story with no Python code. The 5 resolved patches are entirely in the `.j2` template (one line) and the runbook (operator-facing documentation).

### Review Follow-up Pass 2 (2026-05-14)

Both Pass-2 patch findings resolved:

✅ Resolved review finding [Patch — internal contradiction]: `sirmaai-webhook-ingress.md:266` §Failure modes table row 5 — replaced `certbot --nginx --expand` with `certbot certonly --webroot -w /var/www/certbot --expand` to match §Deploy step 4. The failure-modes recovery path now uses the same webroot mode as the original deploy, eliminating the risk of certbot rewriting the nginx config in-place during a TLS-mismatch recovery.

✅ Resolved review finding [Patch — markdown rendering regression]: `www1-rebuild.md:56-59` — reformatted four bare `# NOTE …` lines (which rendered as H1 headings outside the fenced code block) into a blockquote (`> **Note (S04.29):** …`). The note now renders as operator-readable prose between the certbot command and the "Verify:" subsection, matching the meta-commentary intent.

**No new test failures** — this is a pure documentation story. Both fixes are single-surface edits to operator-facing runbook files; no nginx config files or application code were modified.

## Senior Developer Review — Pass 3 (2026-05-14)

**Reviewer:** bmad-code-review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Date:** 2026-05-14
**Verdict:** REVIEW: Approve
**Scope of re-review:** Pass-2 patch closure verification + final adversarial sweep over the four declared S04.29 surfaces (canonical nginx file, .j2 template, new runbook, www1-rebuild.md Phase 2 edit) + AC coverage cross-check.

### Pass-2 patch closure verification

- ✅ **Patch 1 (failure-modes ↔ deploy step 4 contradiction)** — `runbooks/sirmaai-webhook-ingress.md:266` now reads `Re-run \`certbot certonly --webroot -w /var/www/certbot --expand\` with all four \`-d\` flags`. Matches §Deploy step 4 verbatim; the in-place nginx-rewrite risk that Pass 1 Patch 5 was meant to eliminate is no longer reintroduced by the failure-modes recovery path.
- ✅ **Patch 2 (markdown H1-heading regression in DR runbook)** — `runbooks/www1-rebuild.md:53-56` is now a single blockquote (`> **Note (S04.29):** \`-d api.eusolicit.com\` is required so the rebuilt cert covers the SirmaAI webhook ingress from the start. DNS for \`api.eusolicit.com\` must resolve to the new www1 IP BEFORE this step or the ACME challenge will fail. See: \`eusolicit-docs/runbooks/sirmaai-webhook-ingress.md\` §Pre-flight check 1.`). Renders as operator-readable prose between the certbot command and the "Verify:" subsection.

### Final adversarial sweep

1. **Server-block integrity** (canonical + .j2 mirrors): both files carry the byte-identical `:443 ssl` block (server_name, shared cert path, ssl/security snippet includes, log paths, `client_max_body_size 1m`, `location = /webhooks/sirmaai` with `proxy_pass http://127.0.0.1:18004/webhooks/sirmaai` + `proxy_read_timeout 30s` + `proxy_request_buffering on` + `include snippets/proxy-params.conf`, catch-all `location / { return 404; }`). AC 1 + AC 4 satisfied verbatim.
2. **`:80` server_name extension**: canonical L20 + .j2 L16 list all four hosts; .j2 L12 has the per-host `if ($host = api.eusolicit.com)` redirect inserted alphabetically between admin and eusolicit.com — Pass-1 Patch 1 closure verified.
3. **`/ai/` location deletion**: confirmed gone from canonical (former L132-137); .j2 had no such block (acknowledged in Completion Notes). AC 3 satisfied. The pre-deletion grep audit results in Completion Notes (zero hits in `services/` or `frontend/`) are documented evidence that no caller is broken.
4. **Receiver-contract alignment**: response body strings expected by the runbook smoke test (`no_active_subscription`, `invalid_signature`) match `services/sirmaai-gateway/src/sirmaai_gateway/routers/webhooks.py` at lines 185 + 550 + 607 (verified in Pass 2; no regression in Pass 3 diff).
5. **AC 6 (no application code changes)**: confirmed — `git status` on `eusolicit-app` shows the only S04.29-attributable modifications are the two nginx files; all other staged changes are pre-existing in-flight work for S04.27/S04.28 (banner frontend + rate-limit sync). Zero Python touched.
6. **AC 9 (www1-rebuild.md update)**: certbot command extended to four `-d` flags (`www`, root, `admin`, `api`) + blockquote note pointing operators at this story's runbook. DR rebuild Phase 2 will now ship the api.eusolicit.com SAN from the start.
7. **AC 10 (DoD)**: `make lint`/`make type-check` no-op confirmation is honestly documented in Completion Notes (pre-existing sirmaai-gateway failures are NOT attributable to S04.29 which makes zero Python changes); docker-alpine syntax-check limitation is acknowledged; canonical gate is the operator-side `sudo nginx -t` per runbook §Deploy step 3.
8. **AC 11 (cross-tenant invariant)**: correctly delegated to S04.25's existing unit + integration test suite; this story adds the public-facing transport layer only — no regression on the application-layer defense.
9. **AC 12 (status flips)**: sprint-status.yaml row updated; this Pass 3 review flips `review` → `done`-eligible (the operator-side sudo cp + certbot --expand on www1 is explicitly out of scope for the story status flip per AP18-C2 + memory `project_deploy_nginx_manual.md`).

### Deferred items (carry-forward — explicitly NOT blocking S04.29)

The Pass-2 deferred findings remain deferred and out of S04.29 scope:

- Pre-deploy backup overwrite on re-run of §Deploy step 3 (low-probability operator scenario; trivial mitigation).
- Snippet-presence check absent from §Pre-flight (pre-existing pattern across the runbook family).
- Canonical nginx `:80` block uses `$server_name` instead of `$host` (pre-existing; .j2 compensates with per-host `if` blocks — all four now present).
- `.j2` `www.eusolicit.com :443` block diverges from canonical on pre-existing surfaces (`/ws/`, `/health`, `/_next/static/`, dotfile deny) — Story onprem-06 follow-up.
- `certbot --nginx --expand` legacy invocation elsewhere — S04.29 uses `--webroot` correctly; canonical hardening is a separate story.
- S04.25 receiver pre-auth DB lookups + uncapped `webhook-id` header + appendable `X-Forwarded-For` — all pre-existing S04.25 surfaces, not introduced here.

None of these affect the deployable artifacts. They are well-formed follow-up candidates for a future hardening pass.

### Verdict rationale

Both Pass-2 patch findings landed at the claimed locations with the recommended fixes. The infrastructure work (nginx server block, `/ai/` deletion, .j2 mirror, cert-SAN expansion documentation, DR runbook update, smoke-test contract, rollback recipe, failure-modes table) is sound and consistent across the canonical/Ansible-template pair. AC coverage is complete (AC 1–12 all satisfied). The story is ready to flip to `done` pending the standard [PR] post-review gate.

### Acceptance criteria delta vs Pass-2

No new AC failures introduced. The two Pass-2 findings were operator-facing documentation polish and have been resolved without affecting any deployable artifact (nginx config files unchanged since Pass-1 resolved state).

---

## Senior Developer Review — Pass 2 (2026-05-14)

**Reviewer:** bmad-code-review (re-review after Pass 1 patch resolution)
**Date:** 2026-05-14
**Verdict:** REVIEW: Changes Requested
**Scope of re-review:** Pass-1 patch closure verification + adversarial second look at the resolved surfaces.

### Pass-1 patch closure verification

All five Pass-1 patches have landed at the claimed locations:

- ✅ Patch 1 — `.j2` `if ($host = api.eusolicit.com)` redirect is present at `infra/host/templates/nginx-sites/eusolicit.com.conf.j2:12`, alphabetically between `admin` and `eusolicit.com` as recommended.
- ✅ Patch 2 — smoke-test timestamp uses `$(date +%s)` with proper double-quoted header at `runbooks/sirmaai-webhook-ingress.md:156`.
- ✅ Patch 3 — 503 diagnostic is enumerated at `runbooks/sirmaai-webhook-ingress.md:165-169` (cause + `docker compose restart sirmaai-gateway` remediation).
- ✅ Patch 4 — pre-deploy backup `sudo cp … /tmp/eusolicit.com.bak` at line 117 + git-log precondition check at lines 220-233 routing to either git-based or backup-based rollback.
- ✅ Patch 5 — explicit `sudo systemctl reload nginx` to activate the `:80` block at line 130, then `certbot certonly --webroot -w /var/www/certbot --expand` at line 139 (drops `--nginx`'s in-place rewrite mode), with TLS-window caveat documented inline.

The receiver code at `services/sirmaai-gateway/src/sirmaai_gateway/routers/webhooks.py:185,550,607` was sampled to confirm the runbook's expected response bodies (`no_active_subscription`, `invalid_signature`) match the implementation. They do.

### New findings (Pass 2)

#### Patches (in-scope; fix before re-review)

- [x] **[Review][Patch]** Failure-modes table contradicts the new `--webroot` deploy command — `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md:266`. §Deploy step 4 was migrated from `certbot --nginx --expand` to `certbot certonly --webroot -w /var/www/certbot --expand` specifically to prevent certbot from rewriting `/etc/nginx/sites-available/eusolicit.com` in place (the documented motivation for Patch 5). However, the §Failure modes table row 5 still tells an operator hitting a TLS unknown-CA / cert-mismatch incident to "Re-run `certbot --nginx --expand` with all four `-d` flags". An operator following that row would re-introduce exactly the in-place-rewrite drift that Patch 5 was meant to eliminate; on the next `sudo cp` from repo, the certbot-inserted markers would be silently clobbered (or worse, conflict with the canonical file). Fix: replace `certbot --nginx --expand` on line 266 with `certbot certonly --webroot -w /var/www/certbot --expand`, matching §Deploy step 4. This is a single-line fix that closes Patch 5 properly.

- [x] **[Review][Patch]** Phase-2 NOTE block in `www1-rebuild.md` renders as four H1 headings — `eusolicit-docs/runbooks/www1-rebuild.md:56-59`. The S04.29 explanatory note was inserted AFTER the closing ``` of the Phase-2 bash block, with each line starting with `# ` (e.g. `# NOTE (S04.29): \`-d api.eusolicit.com\` is required so …`). In markdown, a leading `# ` outside a fenced code block is an H1 heading, so this produces four giant headings ("NOTE (S04.29): …", "the SirmaAI webhook ingress from the start. …", etc.) between the certbot command and the "Verify:" subsection. Fix options: (a) move the four `#` lines INSIDE the fenced bash block (above the closing ```), or (b) reformat as ordinary prose / a blockquote (e.g. `> **Note (S04.29):** …`). Option (b) is preferred — the message is meta-commentary for the operator, not part of the command, and a blockquote scans better than a code comment.

#### Deferred (low-impact; out of scope for this story)

- [ ] **[Review][Defer]** Pre-deploy backup is overwritten on re-run — `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md:117`. The "Safe to re-run" note at line 185 lists step 3 as idempotent, but a re-run of step 3 (e.g. after a partial failure between steps 3 and 4) overwrites `/tmp/eusolicit.com.bak` with the already-deployed file, losing the true pre-deploy state needed by §Rollback's backup branch. The probability is low (most rollbacks happen after the commit lands, taking the git-based branch) and the operator-side mitigation is trivial (skip the `cp /etc/nginx/...` line on re-run, or rename the backup with a timestamp). Worth adding a one-line guard like `[ -f /tmp/eusolicit.com.bak ] || sudo cp …` in a follow-up runbook polish pass, not blocking for S04.29.

- [ ] **[Review][Defer]** Snippet-presence not checked in §Pre-flight — the deploy assumes `snippets/ssl-params.conf`, `snippets/security-headers.conf`, and `snippets/proxy-params.conf` exist at `/etc/nginx/snippets/`. On a fresh www1 (post-DR rebuild Phase 1), the Ansible playbook `03-nginx-and-certbot.yml` is what installs them; if a hand-edit removed one, §Deploy step 3's `nginx -t` will fail with `open() "snippets/..." failed`. The §Failure modes row 1 ("Hand-edit drift") generically catches this, but a dedicated pre-flight `ls /etc/nginx/snippets/` would make the diagnostic immediate. Pre-existing pattern across the runbook family; defer.

#### Dismissed

- `proxy_pass URI` + `location =` exact match — works as designed; query strings are preserved by default and Standard Webhooks payloads don't carry them.
- Smoke-test webhook-id `smoke-test-12345` collision with a real subscription — vanishingly improbable and the receiver returns 401 (no_active_subscription) for unknown IDs, which is the documented expected response.
- AC 6 sanity — confirmed: the only `eusolicit-app` modifications attributable to S04.29 are the two nginx files; all other staged changes are pre-existing in-flight work for S04.27 (banner) and S04.28 (rate-limit sync), not introduced here.

### Verdict rationale

Two genuine documentation defects survive the Pass-1 resolution: one (Patch finding above) is an internal contradiction that partially undoes Patch 5's intent, the other a markdown rendering regression in the DR runbook. Both are 1–4 lines to fix. The substantive infrastructure work (nginx server block, `/ai/` removal, .j2 mirror, cert-SAN expansion documentation, status flips) is sound; AC coverage remains complete. Requesting one more patch pass to land the two fixes, then this story is ready to flip to `done`.

### Acceptance criteria delta vs Pass-1

No new AC failures introduced. The two findings are operator-facing documentation polish; they do NOT affect the deployable artifacts (nginx config files are correct and unchanged since Pass-1's resolved state).

## Senior Developer Review

**Reviewer:** bmad-code-review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Date:** 2026-05-14
**Verdict:** REVIEW: Changes Requested
**Diff scope:** uncommitted changes on `eusolicit-app` and `eusolicit-docs` repos covering the four declared S04.29 surfaces (canonical nginx file, .j2 template, new runbook, www1-rebuild.md Phase 2 edit) plus the sprint-status row flip.

### Acceptance audit summary

- **AC 1, 2, 3, 4, 5, 7, 9** — fully covered by the implementation. The new `:443` server block contains every required directive byte-for-byte; the `:80` server_name extension is in place; the `/ai/` location is cleanly subtracted from the canonical file (no replacement); the .j2 mirrors the new block byte-identically; the runbook covers every required §-heading with the prescribed bash blocks; `www1-rebuild.md` Phase 2 now lists four `-d` flags with an explanatory note.
- **AC 6 (no application code changes)** — confirmed via `git status` on `eusolicit-app`: the only non-S04.29 modifications are pre-existing in-flight work for S04.27/S04.28 (frontend banner, sirmaai-gateway tests/migrations). S04.29 itself adds zero Python.
- **AC 10 (DoD)** — `make lint`/`make type-check` no-op confirmation and the Docker-based syntax check are both honestly documented in Completion Notes including the alpine-image limitation. Acceptable.
- **AC 11 (cross-tenant invariant)** — correctly delegated to S04.25's existing test suite; this story does not regress that surface.
- **AC 12 (status flips)** — sprint-status.yaml row updated; this review flips `review` → `in-progress` to absorb the patch findings below.

### Findings

#### Patches (in-scope; fix before re-review)

- [x] **[Review][Patch]** `.j2` template HTTP→HTTPS redirect strips api.eusolicit.com host — `eusolicit-app/infra/host/templates/nginx-sites/eusolicit.com.conf.j2:11-17`. The `:80` block uses per-host `if ($host = X) { return 301 https://$host$request_uri; }` for `admin`, `eusolicit.com`, and `www`, then falls through to `location / { return 301 https://$server_name$request_uri; }`. The story added `api.eusolicit.com` to `server_name` (line 15) but did **not** add a parallel `if ($host = api.eusolicit.com)` line. Result: `http://api.eusolicit.com/foo` falls through to the generic `location /` and resolves `$server_name` to the first name in the directive (`www.eusolicit.com`), redirecting to `https://www.eusolicit.com/foo` (which 404s on the gateway path). Fix: insert `if ($host = api.eusolicit.com) { return 301 https://$host$request_uri; }` alongside the existing three `if` lines. (Functional impact is bounded because SirmaAI delivery is HTTPS-direct, but the runbook smoke test or a curl-mistyped HTTP probe will silently land on the wrong vhost.)
- [x] **[Review][Patch]** Smoke-test `webhook-timestamp` is a hardcoded 2024 epoch — `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md:142`. `webhook-timestamp: 1715731200` is 2024-05-14, well outside Standard Webhooks' freshness tolerance (typically ±5 min). The S04.25 receiver may reject with a 4xx code on freshness before reaching signature verification, so the documented "expected 401" outcome will not match reality. Fix: replace the literal with `-H "webhook-timestamp: $(date +%s)"` (or document that operators must substitute a current epoch).
- [x] **[Review][Patch]** Runbook smoke-test expected-responses missing the 503 case — `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md:146-152`. Pre-flight §3 verifies `SIRMAAI_GATEWAY_ENABLED=true` in `.env.prod`, but if the value was changed without a `docker compose restart sirmaai-gateway` the running container will still return 503 from the receiver's first gate. The runbook frames 401 as the only "good" outcome, which would lead the operator to debug nginx instead of the flag/container. Fix: enumerate 503 explicitly with the diagnostic ("flag flipped but container not restarted") in §Deploy step 6 or §Failure modes.
- [x] **[Review][Patch]** Rollback assumes the S04.29 change is committed — `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md:201-213`. The rollback recipe runs `git show <commit-before-S04.29>:eusolicit-app/infra/nginx/eusolicit.com` and `sudo cp` the result. If a rollback is needed before the commit lands (current state — files uncommitted on main), `git show` yields the current file (no S04.29 commit exists), and the rollback is a no-op. Fix: add a precondition check at the top of §Rollback ("Confirm `git log --oneline -- infra/nginx/eusolicit.com | head -3` shows the S04.29 commit; if not, restore from a pre-deploy `/tmp/eusolicit.com.bak` that the operator captured during §Deploy step 3").
- [x] **[Review][Patch]** TLS unknown-CA race between `sudo cp` and `certbot --expand` — `eusolicit-docs/runbooks/sirmaai-webhook-ingress.md:114-129`. After step 3 (`sudo cp` + `nginx -t` + reload), nginx is serving the new `api.eusolicit.com :443` block but the existing cert at `/etc/letsencrypt/live/www.eusolicit.com/fullchain.pem` does NOT yet include the `api.eusolicit.com` SAN (pre-flight §4 confirms this is the expected state for the first run). Any TLS handshake to `api.eusolicit.com` during the window between step 3 and step 5 (post-certbot reload) will present a cert that fails name validation → SirmaAI's delivery agent enters exponential back-off. Fix options (one of): (a) reverse the order — issue/expand the cert first using `--webroot -w /var/www/certbot` (the ACME challenge location is already wired on the `:80` block at canonical L22-24), then `sudo cp` + reload; (b) keep the order but document the brief window in §Deploy and recommend doing the rollout in a low-traffic minute (DNS A-record creation also creates the window). Option (a) is preferred because it avoids the race entirely.

#### Deferred (pre-existing, not introduced by S04.29)

- [x] **[Review][Defer]** Canonical nginx file's `:80` block uses `$server_name` instead of `$host` — `eusolicit-app/infra/nginx/eusolicit.com:26-28`. Pre-existing pattern affecting all four hosts in the `server_name` directive; HTTP requests to `eusolicit.com`, `admin.eusolicit.com`, and (now) `api.eusolicit.com` all redirect to `https://www.eusolicit.com`. The .j2 compensates for three of them with per-host `if` blocks (and the patch above adds the fourth). A separate hardening story should switch the canonical generic `return 301` to `$host` and drop the .j2's `if` block stack.
- [x] **[Review][Defer]** `.j2` template's `www.eusolicit.com :443` block diverges from the canonical file (missing `/ws/`, `/health`, `/_next/static/`, dotfile deny) — `eusolicit-app/infra/host/templates/nginx-sites/eusolicit.com.conf.j2:40-76`. The story onprem-06 contract claimed the two files mirror each other; this is already false on the www block and is the subject of an honest acknowledgement in the Completion Notes. Not introduced by S04.29 but the story's AC 4 wording ("byte-for-byte equivalent on the new server block") is correct only for the new block; a follow-up story should reconcile the www block.
- [x] **[Review][Defer]** `certbot --nginx --expand` may rewrite `/etc/nginx/sites-enabled/eusolicit.com` in place — runbook §Deploy step 4. The `--nginx` plugin can insert managed-by-certbot markers and rewrite `ssl_certificate` paths, creating drift from the repo file. The api block already has correct cert paths so the rewrite is mostly a no-op, but next-deploy's `sudo cp` will silently clobber any annotations. Switching the deploy & rebuild runbooks to `certbot certonly --webroot -w /var/www/certbot --expand` would eliminate the in-place rewrite entirely. Pre-existing pattern (admin.eusolicit.com used the same approach); fix in a hardening story alongside the option-(a) patch above.
- [x] **[Review][Defer]** Pre-auth DB lookup in S04.25 receiver before HMAC verification — `services/sirmaai-gateway/.../routers/webhooks.py`. Three subscription queries run before signature check, which is a DoS amplifier if combined with no nginx rate-limit zone. Out of S04.29 scope (the architecture-amendment §5.1 explicitly accepts no IP allowlist / no edge rate-limit), but worth tracking against S04.25 for a follow-on.
- [x] **[Review][Defer]** `webhook-id` header length is uncapped and used unsanitised as a Redis key — `services/sirmaai-gateway/.../routers/webhooks.py`. Pre-existing S04.25 behavior; cap should be added there, not at the ingress layer.
- [x] **[Review][Defer]** `X-Forwarded-For` propagated via `$proxy_add_x_forwarded_for` is appendable / spoofable — pre-existing `snippets/proxy-params.conf` deployed by `infra/host/playbooks/03-nginx-and-certbot.yml`. Any consumer trusting the first XFF value is exposed; fix at the snippet level affects every vhost and is out of S04.29 scope.

#### Dismissed (noise / false positive / by-design)

- `1m client_max_body_size` is too tight — the story explicitly justifies this against SirmaAI's payload sizing (architecture amendment §5.1); 100× headroom is documented and the choice is intentional.
- `proxy_pass` URI semantics under `location =` — works as designed; nginx replaces the matched URI with the configured upstream path, and `=` precludes the suffix variants.
- Trailing-slash variant `/webhooks/sirmaai/` falls through to 404 — by design (`location =` is exact match); SirmaAI is configured with the literal URL.
- AC 10 syntax-check evidence absent — Completion Notes Task 6 documents the docker-alpine attempt and its limitation; the auditor's claim was based on an incomplete file read.
- `sirmaai-gateway` vs `ai-gateway` diagnostic — CLAUDE.md explicitly documents the temporary alias bridge; the runbook's `docker compose ps sirmaai-gateway` is correct on the post-S04.20 deploy.

### Recommended next step

Apply the five `patch` items as a single follow-up commit on the same branch (none requires more than a few lines), re-run the smoke test against the corrected runbook on a staging vhost if available, and flip the story back to `review`. Architecture and AC coverage are sound; the issues are entirely in the operator-runbook and one .j2 redirect line.
