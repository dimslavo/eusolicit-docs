# Story onprem-06: www1 Itself as Code

Status: ready-for-dev

**Origin:** Onprem pivot decision (eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md). Codifies the host-only state called out in project memory ("www1 host-only state" + "nginx + certbot are manual on www1").
**Priority:** P0 — launch-blocker. Without this, the RTO ≤ 4h commitment is not defensible: a www1 disk loss means "however long it takes the engineer to remember everything configured by hand."
**Epic:** Post-Epic-21 / Onprem Launch
**Points:** 2 | **Type:** infra

## Story

As a **platform-engineering operator**,
I want **the manually-configured state on www1 (authorized_keys, eusolicit-overrides, .env.prod, /etc/nginx, certbot config, debian-user provisioning, firewall rules, docker daemon config) codified as Ansible playbooks or shell scripts under `infra/host/`**,
so that **if the www1 disk is lost or the host needs to be rebuilt, the operator can restore the runtime environment from `git + off-site backups` in ≤4h — matching the RTO promise from ADR-010**.

## Acceptance Criteria

1. **`infra/host/` directory created** (NEW) with the following structure:
   ```
   infra/host/
     README.md                              # operator entry point
     inventory.yaml                          # pins www1 specs (Debian 13, kernel, deps)
     playbooks/
       01-base-system.yml                    # apt update, base packages, NTP, ssh
       02-docker.yml                         # docker engine + compose v2 + daemon.json
       03-nginx-and-certbot.yml              # nginx install + certbot + sites
       04-debian-user.yml                    # user, groups, sudoers, authorized_keys
       05-firewall.yml                       # ufw or iptables rules
       06-eusolicit-overrides.yml            # creates /home/debian/eusolicit-overrides/ skeleton
     templates/
       nginx-sites/                          # nginx site templates (eusolicit.com, etc.)
       env.prod.example                      # template for /home/debian/Projects/.../.env.prod
       env.backup.example                    # template for backup credentials
       authorized_keys.example
   ```
   Format choice: **Ansible** if the team is comfortable; **plain shell scripts** if not. Pick at impl, document.

2. **nginx site config templated** — `/etc/nginx/sites-enabled/eusolicit.com` (the one verified live during pre-flight) is replicated as a template at `infra/host/templates/nginx-sites/eusolicit.com.j2`. Variables: domain names, upstream ports (the `1xxxx` port-collision-avoidance scheme), TLS cert paths.

3. **certbot renewal config** — `infra/host/playbooks/03-nginx-and-certbot.yml` installs certbot + the cert-renewal hook. Includes documentation on initial cert provisioning (`certbot --nginx --expand -d www.eusolicit.com -d eusolicit.com -d admin.eusolicit.com`) since the initial run can't be automated without manual DNS validation.

4. **`debian` user provisioning** — playbook creates the `debian` user with: sudo NOPASSWD for `apt`, `systemctl`, `docker`; SSH access via authorized_keys template; ownership of `/home/debian/Projects/eusolicit/`. Template the authorized_keys file (do NOT commit the real keys; commit an `.example`).

5. **Docker daemon config** — `/etc/docker/daemon.json` codified. Includes: storage driver, log rotation, **storage root moved to `/home/docker/`** (resolves the current `/` partition pressure at 85%).

6. **Firewall rules** — `ufw` or `iptables` rules: allow 22 (SSH), 80 (HTTP), 443 (HTTPS), deny everything else inbound. Templated.

7. **Recovery runbook at `eusolicit-docs/runbooks/www1-rebuild.md`** — step-by-step: "you have a fresh Debian 13 box + git clone of eusolicit-app + off-site backup credentials. Restore service in ≤4h." Includes order-of-operations: base system → docker → eusolicit-overrides skeleton → restore postgres from backup → restore redis from backup → start services → verify health.

8. **Practice recovery on a sacrificial VM** — execute the rebuild runbook end-to-end on a fresh Debian 13 VM (Hetzner Cloud trial or local KVM). Measure wall-clock time. Target: ≤4h. Document result in §First Rebuild Drill Results in the runbook.

## Tasks / Subtasks

- [ ] Task 1: Choose Ansible vs shell; document choice.
- [ ] Task 2: Scaffold `infra/host/` directory.
- [ ] Task 3: Author the 6 playbooks (or shell scripts).
- [ ] Task 4: Template nginx site configs (collect all currently-deployed sites from `/etc/nginx/sites-enabled/` on www1; some are co-tenant — only template the EU Solicit one).
- [ ] Task 5: Codify Docker daemon.json + storage-root move (CAREFUL — moving storage root requires docker stop + data migration; treat as a separate sub-task on www1 with a maintenance window).
- [ ] Task 6: Author `www1-rebuild.md` runbook.
- [ ] Task 7: Run the practice rebuild on a sacrificial VM; measure RTO; refine playbooks.
- [ ] Task 8: Commit `.example` templates only; document where real credentials live (host-only).

## Dev Notes

### Currently-manual state on www1 (from project memory + live inspection)

Per memory entry "www1 host-only state": these are NOT in the repo today:
- `authorized_keys` for the `debian` user
- `/home/debian/eusolicit-overrides/` (docker-compose.prod.override.yml + .env.prod + likely the future .env.backup)
- `/etc/nginx/` configuration
- certbot's `/etc/letsencrypt/`

Per memory entry "nginx + certbot are manual on www1":
> "deploy.yml never touches /etc/nginx; infra/nginx/ edits need a sudo cp on www1"

The recent commit `3cba326 fix(nginx): strip /admin-api/ prefix before forwarding to admin-api` updated the nginx config in the repo at `infra/nginx/eusolicit.com.conf` (verified live: this is the canonical source). But applying it to `/etc/nginx/sites-enabled/eusolicit.com` requires manual `sudo cp` on www1. This story doesn't change that workflow — it just makes the manual step reproducible.

### Why this is launch-blocking

Without `infra/host/`:
- A disk loss requires the engineer to remember nginx config, ESO secrets, env files, ufw rules, docker daemon config — items that took weeks of trial-and-error to converge.
- The RTO ≤ 4h promise is fiction. Realistic recovery time without `infra/host/` is "days to whenever-the-engineer-remembers-everything."
- The Trust Center disclosure ("RTO ≤ 4h, RPO ≤ 24h") becomes false advertising.

### Why Ansible vs shell

Ansible:
- Idempotent by default; safer to re-run.
- Inventory makes multi-host expansion easy (relevant for hypothetical Phase-2 standby host).
- Standard tool; new operators understand it.

Shell:
- Zero new tool deps; works with what's already on www1.
- Faster to author for a team unfamiliar with Ansible.
- Idempotency is the author's responsibility (script defensively).

Recommend Ansible for this scope. The host-as-code investment pays back over time.

### References

- ADR-010 rewrite: `architecture.md` §ADR-010 (2026-05-11)
- Decision record: `onprem-pivot-decision-2026-05-11.md`
- Project memory: "www1 host-only state" + "nginx + certbot are manual on www1"
- nginx config (canonical source in repo): `eusolicit-app/infra/nginx/eusolicit.com.conf` (where the latest commit lives — must be `sudo cp`ed to `/etc/nginx/sites-enabled/` manually today)
- Existing co-tenant nginx sites (visible on www1): `agenticsai.endigitalx.com`, `celthrac.com`, `crewai.endigitalx.com`, `crew.celthrac.com`, `dev.ai.celthrac.com`, `endigital.io`, `eusolicit.com`, `lk.agenticsai.endigitalx.com`, `n8n.endigitalx.com`

### Out of scope

- Multi-host orchestration (single www1 is the scope).
- Bare-metal provisioning (assume Debian 13 boots; we codify from there up).
- Co-tenant playbooks (those are other tenants' concern).
- Active/standby second host (future Phase-2 epic).
