# lg-buddy

Keep an LG WebOS TV in sync with a Linux host's power and session state. Bash CLI + systemd units — no compiled artifacts.

## Overview

Two scripts in `bin/`:

- **`lg-buddy`** — the single entry point called by every trigger. Talks to the TV over the WebOS WebSocket API via `bscpylgtv`; sends Wake-on-LAN packets for cold-start. Four subcommands: `on`, `off`, `sync-input`, `status`.
- **`lg-buddy-unlock-listen`** — a long-running daemon (user systemd service) that monitors the freedesktop `ScreenSaver.ActiveChanged` D-Bus signal and calls `lg-buddy on` on unlock. Covers the idle-lock / unlock path, which is not a power event and therefore not handled by the sleep/resume hooks.

## Commands

```bash
lg-buddy on          # wake the TV (WOL + wait + set_input); idempotent if already on
lg-buddy off         # power off; skip if host is rebooting
lg-buddy sync-input  # set input only if TV is already on
lg-buddy status      # print power state; exit 1 if TV is unreachable
```

Config is read from `/etc/lg-buddy/lg-buddy.env` (override with `LG_BUDDY_ENV=`). See `lg-buddy.env.example` for all knobs (`TV_IP`, `TV_MAC`, `TV_INPUT`, `PAIRING_DB`, `BSCPYLGTV`, `WOL_COUNT`, `INPUT_RETRIES`).

## Install

### Ansible (preferred for homelab)

Deployed by the `desktop-common` Ansible role in `repos/homelab-ansible`, gated on `desktop_install_lg_buddy: true` in host vars.

### Manual / tarball

Each tagged release ships `lg-buddy-<version>.tar.gz` (+ `.sha256`) as a GitHub Release asset, bundling `bin/`, `systemd/`, `lg-buddy.env.example`, and `README.md`.

```bash
# 1. Fetch and verify the tarball (substitute actual version)
curl -fsSL -O https://github.com/jedwards1230/lg-buddy/releases/download/v<version>/lg-buddy-<version>.tar.gz
curl -fsSL -O https://github.com/jedwards1230/lg-buddy/releases/download/v<version>/lg-buddy-<version>.tar.gz.sha256
sha256sum -c lg-buddy-<version>.tar.gz.sha256
tar -xzf lg-buddy-<version>.tar.gz

# 2. Install files
sudo cp -a lg-buddy-<version>/bin/* /opt/lg-buddy/bin/
sudo python3 -m venv /opt/lg-buddy/venv && sudo /opt/lg-buddy/venv/bin/pip install bscpylgtv
sudo mkdir -p /etc/lg-buddy
sudo cp lg-buddy-<version>/lg-buddy.env.example /etc/lg-buddy/lg-buddy.env
# Edit /etc/lg-buddy/lg-buddy.env with TV_IP, TV_MAC, TV_INPUT, PAIRING_DB

# 3. Install + enable systemd units
sudo cp lg-buddy-<version>/systemd/lg-buddy.service lg-buddy-wake.service /etc/systemd/system/
sudo cp lg-buddy-<version>/systemd/system-sleep/lg-buddy /lib/systemd/system-sleep/
sudo chmod +x /lib/systemd/system-sleep/lg-buddy
sudo systemctl daemon-reload
sudo systemctl enable --now lg-buddy.service lg-buddy-wake.service

# 4. Enable the user unit inside the graphical session
cp lg-buddy-<version>/systemd/user/lg-buddy-unlock.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now lg-buddy-unlock.service
```

## Architecture

| Component | Trigger | Action |
|---|---|---|
| `lg-buddy.service` (system) | boot / shutdown | `lg-buddy on` / `lg-buddy off` |
| `lg-buddy-wake.service` (system) | resume from any sleep state | `lg-buddy on` |
| `systemd/system-sleep/lg-buddy` (sleep hook) | pre-suspend/hibernate | `lg-buddy off` |
| `lg-buddy-unlock.service` (user) | session unlock (D-Bus signal) | `lg-buddy on` |

The `on` path: if TV is already active, re-select input only. If off, spray `WOL_COUNT` WOL packets (2 s apart), wait 6 s for the panel to boot, then retry `set_input` up to `INPUT_RETRIES` times (re-sending WOL on each retry). The `off` path checks `reboot.target` via `journalctl` and is a no-op if the host is rebooting.

## Conventions and Gotchas

- **WebOS 1008 throttle**: rapid successive `bscpylgtv` connections trip WebOS error `1008 EWS Try Again Later`, which does **not** self-clear. Space out calls; avoid any retry loop that reconnects without adequate backoff. This is why `cmd_on` sleeps between WOL packets and before the first `set_input` attempt.
- **Pairing DB**: keep `PAIRING_DB` on a partition that survives an OS-drive wipe (e.g. `/mnt/nvme1/lg-buddy/pairing.db`). If the file is lost, the TV will re-prompt for pairing on the next connect.
- **Session unlock vs power events**: idle screen-lock and unlock are NOT power/resume events — `lg-buddy-wake.service` and the sleep hook never fire for them. `lg-buddy-unlock-listen` exists specifically to cover this gap via D-Bus `ScreenSaver.ActiveChanged`.
- **Config is gitignored**: `*.env` is in `.gitignore` (with `!*.env.example` exception). Never commit actual credentials or IP/MAC addresses.
- **No build step**: pure Bash — no compilation, no test suite, no linter CI. The PR review workflow (`claude-pr-review.yml`) is the quality gate.
- **Release**: opt-in — label a merged PR `semver:patch`, `semver:minor`, or `semver:major` to trigger a release. No label → no release. The release workflow builds and attaches the install tarball.
