# Runbook: Clock Drift / NTP

**Trigger:** `HostClockDriftHigh` (abs(node_timex_offset_seconds) > 30s)
**Story:** onprem-04 | **SLO:** platform
**SLA-Scope**: in-scope

## Symptoms

Clock drift breaks:
- Backup timestamps (off-site retention windows skewed).
- Audit log timestamps (forensic ordering unreliable).
- Alembic migration timing (compared against `last_updated_by` in project-context).
- TLS cert validity windows (cert renewal failures).
- JWT issued-at / expires-at validation (auth flakiness).

## Triage

```bash
# What is the actual drift?
timedatectl

# Is the time sync service running?
systemctl status systemd-timesyncd 2>&1 | head -10
# OR (if using chrony / ntpd)
chronyc tracking 2>&1 || true

# How far off are we from a known-good source?
sudo ntpdate -q pool.ntp.org 2>&1 || true
```

## Resolution

```bash
# 1. Force an immediate sync
sudo systemctl restart systemd-timesyncd

# 2. If systemd-timesyncd is not enabled (unusual)
sudo timedatectl set-ntp true
sudo systemctl enable --now systemd-timesyncd

# 3. Verify sync re-established
timedatectl
# 'System clock synchronized: yes' is what we want.
```

If drift persists:

- Firewall may block UDP/123 outbound. Check `sudo ufw status` and provider firewall.
- Internal NTP server may be unreachable; switch to public pool:
  ```bash
  sudo bash -c 'cat > /etc/systemd/timesyncd.conf <<EOF
  [Time]
  NTP=pool.ntp.org
  FallbackNTP=time.cloudflare.com
  EOF'
  sudo systemctl restart systemd-timesyncd
  ```

### Long-term

NTP setup is captured in the `onprem-06-www1-itself-as-code` playbook `01-base-system.yml`.

## Verification

```bash
# Synchronized, offset back under the 30s alert threshold
timedatectl
# Expect: 'System clock synchronized: yes' and 'NTP service: active'

sudo ntpdate -q pool.ntp.org 2>&1 || true
```

`HostClockDriftHigh` clears once `abs(node_timex_offset_seconds)` falls back under 30s for the alert window.

## Rollback

Forcing a time sync is **forward-only** — a corrected clock is the goal; there is nothing to undo. The one reversible change is editing `/etc/systemd/timesyncd.conf` (the "if drift persists" step): if switching to the public pool causes issues (e.g. policy requires the internal NTP server), restore the previous file contents and `sudo systemctl restart systemd-timesyncd`. Back up the file before editing if the original config is not already under `onprem-06` host-as-code.

## Related

- Alert rule: `infra/observability/prometheus/rules/host-alerts.yaml`
- onprem-06: host-as-code playbooks (`01-base-system.yml`)
