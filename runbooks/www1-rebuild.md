# Runbook: www1 Rebuild from Scratch (Story onprem-06)

**Audience:** on-call engineer executing a full-host disaster recovery
**RTO target:** ≤ 4h | **RPO target:** ≤ 24h (off-site backup interval)
**Prerequisite:** off-site backups exist + accessible; `infra/host/` playbooks applied successfully on the new host

## When to use

The www1 host itself is lost — disk failure, DC fire, irrecoverable misconfiguration. NOT for partial recoveries (use `postgres-restore.md`, `redis-restore.md`, or `docker-daemon-recovery.md` for those).

## Pre-rebuild (≤ 30 min)

1. Declare incident (S1). Page on-call.
2. Confirm www1 is truly unrecoverable. Snapshot any forensic artifacts if possible.
3. Provision a fresh Debian 13 host (Hetzner Cloud, Hetzner Dedicated, or equivalent). Note the new IP.
4. Get SSH access. Drop your bootstrap key into `/root/.ssh/authorized_keys` (cloud-init or provider console).
5. Test SSH: `ssh root@<new-ip>`.

## Phase 1 — host-as-code apply (≤ 30 min)

```bash
# On a workstation with Ansible installed AND this repo cloned
cd /path/to/eusolicit-app/infra/host

# Update inventory.yaml: temporarily point www1.endigitalx.com at the new IP
# OR add an explicit ansible_host override:
#   ansible_host: <new-ip>

# Dry-run
ansible-playbook -i inventory.yaml playbooks/site.yml --check --diff

# Apply
ansible-playbook -i inventory.yaml playbooks/site.yml
```

What this installs:
- Base system (apt, NTP, sysctl hardening)
- Docker + compose plugin with storage-root on `/home/docker`
- nginx + certbot (cert renewal hook ready)
- `debian` user with NOPASSWD sudo for op commands
- ufw firewall (22 / 80 / 443 inbound)
- `/home/debian/eusolicit-overrides/` skeleton + log/backup dirs
- Cron file at `/etc/cron.d/eusolicit-backup`

## Phase 2 — first-run cert provisioning (≤ 10 min)

Certbot's initial run is interactive (email + ToS accept). One-time:

```bash
ssh debian@<new-ip>
sudo certbot --nginx --expand \
  -d www.eusolicit.com -d eusolicit.com \
  -d admin.eusolicit.com -d api.eusolicit.com \
  --email <ops-email> --agree-tos --no-eff-email
```
> **Note (S04.29):** `-d api.eusolicit.com` is required so the rebuilt cert covers
> the AgenticSAI webhook ingress from the start. DNS for `api.eusolicit.com` must
> resolve to the new www1 IP BEFORE this step or the ACME challenge will fail.
> See: [`eusolicit-docs/runbooks/agenticsai-webhook-ingress.md`](agenticsai-webhook-ingress.md) §Pre-flight check 1.

Verify:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Phase 3 — repopulate host-only credentials (≤ 30 min)

These are NOT in git. Sources to recover from: password manager, offline encrypted backup, organizational secrets store.

```bash
# 1. .env.prod (application secrets)
vim /home/debian/eusolicit-overrides/.env.prod
chmod 600 /home/debian/eusolicit-overrides/.env.prod
# at minimum: POSTGRES_PASSWORD, STRIPE_*_KEY, JWT signing keys

# 2. .env.backup (Hetzner Storage Box for restic)
cp /home/debian/eusolicit-overrides/.env.backup.example /home/debian/eusolicit-overrides/.env.backup
vim /home/debian/eusolicit-overrides/.env.backup
chmod 600 /home/debian/eusolicit-overrides/.env.backup
# at minimum: RESTIC_REPOSITORY, RESTIC_PASSWORD

# 3. .env.alertmanager (paging — email + Telegram)
cp /home/debian/eusolicit-overrides/.env.alertmanager.example /home/debian/eusolicit-overrides/.env.alertmanager
vim /home/debian/eusolicit-overrides/.env.alertmanager
chmod 600 /home/debian/eusolicit-overrides/.env.alertmanager

# 4. Secret files
echo 'YOUR_SMTP_PASSWORD'   > /home/debian/eusolicit-overrides/secrets/smtp_password
echo 'YOUR_SLACK_WEBHOOK'   > /home/debian/eusolicit-overrides/secrets/slack_webhook
chmod 600 /home/debian/eusolicit-overrides/secrets/*

# 5. docker-compose.prod.override.yml — if any host-specific overrides exist
# (e.g., port remap for co-tenants). On a clean host this may be empty.
```

## Phase 4 — clone app + restore data (≤ 2h)

```bash
# Clone
mkdir -p /home/debian/Projects/eusolicit
cd /home/debian/Projects/eusolicit
git clone git@github.com:dimslavo/eusolicit-app.git
git clone git@github.com:dimslavo/eusolicit-docs.git

# Bring up postgres + redis only (so we have somewhere to restore into)
cd eusolicit-app
docker compose -f docker-compose.prod.yml up -d postgres redis

# Wait for postgres healthy
docker compose -f docker-compose.prod.yml ps postgres
sleep 30

# Restore postgres (this is the long step — depends on DB size)
bash scripts/onprem/postgres-restore.sh --to-prod
# Type 'I-UNDERSTAND' at the prompt. ETA: ~30 min for typical DB size.

# Restore redis (fast)
bash scripts/onprem/postgres-restore.sh --to-prod   # OOPS: typo, the redis one:
bash scripts/onprem/redis-restore.sh --to-prod
# Type 'I-UNDERSTAND' at the prompt.

# Bring up the full app stack
bash scripts/deploy.sh
```

## Phase 5 — DNS flip + verify (≤ 30 min)

```bash
# Update DNS A records to point at the new IP. Provider varies (Cloudflare,
# Route 53, registrar DNS, etc.).

# Wait for DNS propagation
dig +short www.eusolicit.com
# Should return new IP. May take 5-60 min depending on TTL.

# Verify external reach
curl -sLf https://www.eusolicit.com | grep -qi "EU Solicit" && echo "public OK"
curl -sf https://www.eusolicit.com/admin-api/healthz

# All 6 services healthy locally
for port in 18001 18002 18003 18004 18005 18007; do
  curl -sf http://127.0.0.1:$port/healthz && echo " port $port OK" || echo " port $port FAIL"
done
```

## Phase 6 — observability + cron sanity (≤ 30 min)

```bash
# Bring up observability stack
cd /home/debian/Projects/eusolicit/eusolicit-app/infra/observability
docker compose -f docker-compose.observability.yml \
  --env-file /home/debian/eusolicit-overrides/.env.alertmanager up -d

# Verify cron file is in place + crond running
ls /etc/cron.d/eusolicit-backup
systemctl status cron

# Trigger one backup to verify the chain works
bash /home/debian/Projects/eusolicit/eusolicit-app/scripts/onprem/postgres-backup.sh
```

## Phase 7 — post-mortem (≤ 24h after restoration)

Document in `eusolicit-docs/post-mortems/`:
- What failed on www1 (root cause)
- Exact timeline (T+0 = first user impact)
- Wall-clock from incident open → service restored
- Where the rebuild plan was tested vs not
- What slowed us down (record this in §First Rebuild Drill Results below)

## §First Rebuild Drill Results

| Date | Source of failure | Wall-clock RTO | Issues encountered | Notes |
|---|---|---|---|---|
| TBD | (sacrificial-VM drill or real incident) | — | — | (Fill in on first practice rebuild — required to close Story onprem-06 AC8.) |

Target: ≤ 4h wall-clock. Anything longer points to runbook gaps; iterate the playbooks.

## References

- Story: `eusolicit-docs/implementation-artifacts/onprem-06-www1-itself-as-code.md`
- Playbooks: `eusolicit-app/infra/host/playbooks/`
- Postgres restore: `runbooks/postgres-restore.md`
- Redis restore: `runbooks/redis-restore.md`
- Pivot decision: `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md`
