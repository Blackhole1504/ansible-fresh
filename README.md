# ansible-fresh

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL%20v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![lint](https://github.com/Blackhole1504/ansible-fresh/actions/workflows/lint.yml/badge.svg)](https://github.com/Blackhole1504/ansible-fresh/actions/workflows/lint.yml)

Baseline security hardening for fresh **Ubuntu 24.04 / 26.04 LTS** servers,
fetched and executed by **cloud-init** via `ansible-pull` from a git repo.

```
cloud-init ──▶ ansible-pull (git clone) ──▶ local.yml ──▶ 9 hardening roles
                                          └▶ +2 optional roles (toggles, default off)
```

## What it enforces

| Area | Result |
|---|---|
| SSH | Key-only auth, `PermitRootLogin no`, forwarding off, CIS-grade algorithms |
| Sudo | Passwordless sudo for exactly one detected admin user; cloud-init's blanket grant removed |
| Kernel | sysctl hardening: no routing/redirects, kptr/dmesg restrict, BPF hardening |
| Updates | unattended-upgrades (security origin), daily apt timers |
| Firewall | UFW default-deny incoming, SSH allowlisted (with lockout guards) |
| fail2ban | sshd jail via systemd journal, bans through UFW |
| Audit | auditd with a minimal CIS-derived ruleset |
| Time | systemd-timesyncd with configurable NTP pool |
| Services | rpcbind/cups disabled; snapd removed (toggle) — AppArmor unaffected |
| Docker *(off)* | `docker.io` + `docker-compose-v2`, json-file logs rotated 10m×3, live-restore |
| Proxy *(off)* | nginx installed, default site removed, 80/443 auto-allowed in UFW |

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
Two roles are opt-in via toggles there (`docker_host_enabled`,
`reverse_proxy_enabled`, both default `false`); if the sudo admin user
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

## Snapd removal & AppArmor

Removing snapd does **not** affect AppArmor (different packages; asserted
every run via `aa-status --enabled`). Keep snap with
`common_disable_snapd: false`.

## Docker / reverse proxy later

With both toggles off, the baseline installs neither — but adds **no kernel
rule that would block them later**:

- `net.ipv4.ip_forward = 0` is written by the sysctl role, but `dockerd`
  sets it to 1 at daemon start, so Docker works regardless.
- `rp_filter` is absent from the sysctl drop-in (a strict value breaks
  Docker bridge networking; Ubuntu's loose default is docker-safe).
- Flipping `docker_host_enabled`/`reverse_proxy_enabled` to `true` opens
  UFW ports (`80`/`443`) and sets `ip_forward=1` in the same run.

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
  docker_host/  reverse_proxy/          # optional (toggles, default off)
```

## Limitations

- Ubuntu only (asserted at run time)
- No user creation — the cloud-init/Proxmox `--ciuser` user is kept, and it
  holds the sole passwordless-sudo grant
- Not a full CIS benchmark — a sane baseline, not certification

## License

AGPL-3.0 — see [`LICENSE`](LICENSE).
