# lg-buddy

Keep an LG WebOS TV in sync with a Linux host's power and session state.
Built for a desktop whose only display is the TV: wake + select the input
when the host comes up, power off when it goes down — and, crucially, wake
the TV again when you unlock an already-running session (an idle screen lock
is not a power event, so nothing else covers that).

Control is over the WebOS WebSocket API via
[`bscpylgtv`](https://github.com/chros73/bscpylgtv), with Wake-on-LAN for the
cold-start path.

## One command, many triggers

Everything routes through a single config-driven script:

```
lg-buddy on          wake the TV, or just set the input if it's already on
lg-buddy off         power off (skipped automatically when the host reboots)
lg-buddy sync-input   switch input only, if the TV is already on
lg-buddy status      print the TV power state
```

| Trigger | Calls |
|---|---|
| `lg-buddy.service` (boot / shutdown) | `on` / `off` |
| `lg-buddy-wake.service` (resume from any sleep state) | `on` |
| `system-sleep/lg-buddy` (pre-suspend/hibernate) | `off` |
| `lg-buddy-unlock.service` (user session, on unlock) | `on` |

`on` is idempotent: if the TV is already on it only re-selects the input;
if it's off it sends `WOL_COUNT` wake packets, waits for the panel to boot,
then retries `set_input` up to `INPUT_RETRIES` times (re-sending WOL each
time in case a packet was dropped mid-boot). `off` checks `reboot.target`
first and leaves the TV on through a reboot.

## Install

`bin/` → `/opt/lg-buddy/bin/`, a `bscpylgtv` venv at `/opt/lg-buddy/venv`,
and config at `/etc/lg-buddy/lg-buddy.env` (see `lg-buddy.env.example`).
Install the units from `systemd/` and enable:

```
systemctl enable --now lg-buddy.service lg-buddy-wake.service
systemctl --user enable --now lg-buddy-unlock.service   # in the graphical session
```

This is deployed in the homelab via the `desktop-common` Ansible role
(gated by `desktop_install_lg_buddy`).

Each tagged release also ships a `lg-buddy-<version>.tar.gz` install tarball
(plus a `.sha256` checksum) bundling `bin/`, `systemd/`, `lg-buddy.env.example`,
and this README — a self-contained alternative to the Ansible install. Verify it
with `sha256sum -c lg-buddy-<version>.tar.gz.sha256`, unpack, and follow the
steps above.

## Config

| Var | Purpose |
|---|---|
| `TV_IP` / `TV_MAC` | TV address + MAC for WOL |
| `TV_INPUT` | input to select after wake (e.g. `HDMI_2`) |
| `PAIRING_DB` | WebOS pairing token DB (keep on a wipe-surviving partition) |
| `BSCPYLGTV` | path to the `bscpylgtv` CLI |
| `WOL_COUNT` / `INPUT_RETRIES` | cold-start tuning |
