# ansible-fresh

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL%20v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![lint](https://github.com/Blackhole1504/ansible-fresh/actions/workflows/lint.yml/badge.svg)](https://github.com/Blackhole1504/ansible-fresh/actions/workflows/lint.yml)

Baseline security hardening for fresh **Ubuntu 24.04 / 26.04 LTS** servers,
fetched and executed by **cloud-init** via `ansible-pull` from a git repo.

```
cloud-init ──▶ ansible-pull (git clone) ──▶ local.yml ──▶ 9 hardening roles
```

## What it enforces

| Area | Result |
|---|---|
| SSH | Key-only auth, `PermitRootLogin no`, forwarding off, CIS-grade algorithms |
| Sudo | Passwordless sudo for exactly one detected admin user; cloud-init's blanket grant removed |
| Kernel | sysctl hardening: no routing/redirects, kptr/dmesg restrict, BPF hardening |
| Updates | unattended-upgrades (security origin) in a nightly 22:00–05:40 window — update+upgrade back-to-back; conditional auto-reboot 06:00–06:59 |
| Firewall | UFW default-deny incoming, SSH allowlisted (with lockout guards) |
| fail2ban | sshd jail via systemd journal, bans through UFW |
| Audit | auditd, minimal CIS-derived ruleset; age-only daily rotation kept 180 days, halt under 1 GB free |
| Time | systemd-timesyncd with configurable NTP pool |
| Services | rpcbind/cups disabled; snapd removed (toggle) — AppArmor unaffected |

## Quickstart (Proxmox VE)

1. Push this repo to `https://github.com/Blackhole1504/ansible-fresh`.
   (It already lives there — fork it and repoint the URLs below at your
   fork if you want your own baseline.)
2. Upload [`cloud-init/vendor-config.yaml`](cloud-init/vendor-config.yaml) to a
   snippets storage (e.g. `local`) on your PVE node — `pvesm upload local
   snippets vendor-config.yaml` — and edit its `runcmd` URL to point at your
   fork. Snippets storage must exist on every node the VM can migrate to.
3. Create the VM from an Ubuntu 24.04/26.04 cloud-image template with a
   cloud-init drive, then:

   ```
   qm set <vmid> --ciuser <name> --sshkey ~/.ssh/id_ed25519.pub \
     --cicustom "vendor=local:snippets/vendor-config.yaml"
   ```

   `--ciuser`/`--sshkey` are the source of truth for identity: cloud-init
   creates that user, and the playbook detects it (see *Lockout safety*).
4. Start the VM. cloud-init installs `ansible` + `git`, upgrades packages
   (PVE's generated user-data does it — the vendor config has no `package_*`
   keys to avoid upgrading twice), then runs `ansible-pull` against the repo
   as root. First boot takes a few minutes.

Verify after boot:

```
ssh <name>@<server>        # login banner: 'ansible-fresh: baseline hardening applied successfully.'
cloud-init status --wait   # → 'status: done'
sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
sudo -l                    # → '(root) NOPASSWD: ALL' for <name> only
sudo grep -rn NOPASSWD /etc/sudoers.d
sudo ufw status verbose
sudo fail2ban-client status sshd
```

## Manual run / updates

`ansible-pull` always fetches the latest revision before running, so
re-running this command is the way to apply repo updates to an existing
server:

```
sudo apt update && sudo apt install -y ansible git
sudo ansible-pull -U https://github.com/Blackhole1504/ansible-fresh.git \
  --clean --accept-host-key -e @/etc/ansible/ansible-pull-extra-vars.yaml -v
```

Or, from a dev checkout: `git pull --ff-only && sudo ansible-playbook local.yml`.
Every role is idempotent — a second run should report zero changes. Lint:
`ansible-lint` and `yamllint` (config at repo root).

## Configuration

All tunables live in **[`group_vars/all.yml`](group_vars/all.yml)** — the
single source of truth, commented per role. Per-deployment overrides: edit
the `write_files` block in the vendor config or pass `-e @your-vars.yml`.
If the sudo admin user
auto-detection ever fails or is ambiguous, set `sudo_admin_user` explicitly.

## Lockout safety

Three independent guards, by design:

1. **ssh_hardening** asserts an `authorized_keys` file exists under `/home`
   or `/root` before disabling password auth.
2. **firewall** asserts the SSH port is allowlisted before enabling
   default-deny.
3. **sudo_hardening** resolves exactly one admin user before touching the
   sudo policy (override → `SUDO_USER` → cloud-init sudoers grant →
   previous drop-in → cloud-init user-data; full order in the role's
   tasks file). Ambiguity
   or absence fails the run, and a final audit asserts no NOPASSWD grant
   remains for anyone else. Fix: set `sudo_admin_user`.

The sshd drop-in is validated with `sshd -t` before reload, sudoers changes
with `visudo -cf`, and the firewall role self-pings the host after enabling
UFW. Keep your current session open until a **new** SSH connection works.

## Audit log retention

Audit logs rotate **by age only**: auditd's own size/number rotation is
disabled and a daily timer (`ansible-auditd-archive.timer`) runs logrotate
with `copytruncate` — no daemon restart needed. `auditd_retention_days`
plain-text daily files are kept (180 ≈ 6 months, well under 200 MB at this
ruleset's volume) and stay directly searchable:

```
sudo ausearch -if /var/log/audit/audit.log.1 -k identity
```

Disk guards: syslog warning at 1.5 GB free, system halt at 1 GB — audit
fails closed. Caveat: `copytruncate` can lose events written in the
copy/truncate instant (accepted; the alternative fights auditd's
`RefuseManualStop` lock).

## Snapd removal & AppArmor

Removing snapd does **not** affect AppArmor (different packages; asserted
every run via `aa-status --enabled`). Keep snap with
`common_disable_snapd: false`.

## Nightly updates & reboots

One ansible-managed timer (`ansible-apt-nightly.timer`) fires inside a
randomized **22:00–05:40** window and runs apt's daily script with no
argument — so `apt-get update` and `unattended-upgrade` execute
back-to-back in a single run (zero gap between refresh and install). The
stock `apt-daily`/`apt-daily-upgrade` timers are stopped and disabled;
`APT::Periodic` intervals are set to `always` so the custom schedule is
authoritative (masking the stock timers also silences the
`apt-news`/`esm-cache` motd sidekicks — cosmetic).

Only **security-pocket** origins are auto-installed (`-security`, plus
ESM pockets when Ubuntu Pro is attached); `-updates` is never touched.
Check on a live host:

```
sudo unattended-upgrade --dry-run --debug 2>&1 | grep 'Allowed origins'
systemctl cat ansible-apt-nightly.timer && systemctl list-timers ansible-apt-nightly.timer
```

If the run leaves `/var/run/reboot-required`, the machine reboots in the
**06:00–06:59** slot; the minute is derived from the host FQDN hash, so
machines spread across the hour instead of rebooting in lockstep. The
reboot is skipped when any user is logged in at scheduling time (checked
during the nightly run — logins after scheduling do not block it), and
new logins are refused in the last 5 minutes before it (`/run/nologin`).
Reboots happen **without confirmation**. `Persistent=true` means a
machine powered off overnight runs the missed window at next boot.
Worst case: a run that finishes after its reboot time (e.g. started
05:40 and overran) reboots immediately on completion.

## Docker compatibility

This baseline adds no kernel rule that would block a Docker install:

- `net.ipv4.ip_forward = 0` is written by the sysctl role, but `dockerd`
  sets it to 1 at daemon start, so Docker works regardless.
- `rp_filter` is absent from the sysctl drop-in (a strict value breaks
  Docker bridge networking; Ubuntu's loose default is docker-safe).

Note: Docker bypasses UFW for published container ports (`-p 8080:80` is
open even under default-deny) — restrict published ports at the Docker
level or front them with the reverse proxy.

## Dependencies

- `ansible-core` ≥ 2.16, as provided by the full **`ansible`** apt package
  (not `ansible-core` alone) — it bundles `community.general` (used for
  `ufw` and `timezone` modules). Ubuntu 24.04 ships that floor; 26.04 newer.
- No external roles/collections to install (`requirements.yml` is empty);
  `ansible-pull` runs with zero `ansible-galaxy` steps.

## Repo layout

```
local.yml                    entrypoint (ansible-pull default)
ansible.cfg                  self-contained config
inventory/hosts              localhost only
group_vars/all.yml           ALL tunables
cloud-init/vendor-config.yaml  Proxmox cicustom vendor slot
roles/
  common/  timesync/  ssh_hardening/  sudo_hardening/  sysctl_hardening/
  unattended_upgrades/  firewall/  fail2ban/  auditd/
```

## Limitations

- Ubuntu only (asserted at run time)
- No user creation — the cloud-init/Proxmox `--ciuser` user is kept, and it
  holds the sole passwordless-sudo grant
- Not a full CIS benchmark — a sane baseline, not certification

## License

AGPL-3.0 — see [`LICENSE`](LICENSE).
