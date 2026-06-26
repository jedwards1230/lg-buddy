# Contributing to lg-buddy

`lg-buddy` keeps an LG WebOS TV in sync with a Linux host's power and session state — a pure Bash CLI plus systemd units, no compiled artifacts. All changes go through the workflow below.

## Prerequisites

- `bash`
- `python3` with [`bscpylgtv`](https://github.com/chros73/bscpylgtv) — the WebOS control library the scripts shell out to
- `wakeonlan` (cold-start WOL) and `gdbus` (the unlock listener's D-Bus monitor) — both invoked directly by the scripts
- `systemd` — to exercise the units and the sleep hook
- A reachable LG WebOS TV with a saved pairing DB (and a running graphical session, for the unlock path) to exercise behavior end-to-end
- `shellcheck` (optional, recommended) — to lint the scripts locally

## Build, test & lint

This is pure Bash — there is no build step, test suite, or lint job in CI; the automated PR review is the quality gate. Before pushing, lint the scripts and smoke-test the paths your change touches. Point `LG_BUDDY_ENV` at your own env file — the installed `/etc/lg-buddy/lg-buddy.env` won't exist until after a deploy:

```bash
# Lint the shell scripts locally (recommended; not run in CI)
shellcheck bin/lg-buddy bin/lg-buddy-unlock-listen

# Copy the example and fill in your TV's details (TV_IP, TV_MAC, TV_INPUT, PAIRING_DB)
cp lg-buddy.env.example /tmp/lg-buddy.env
export LG_BUDDY_ENV=/tmp/lg-buddy.env

# Exercise the subcommand(s) your change affects
bin/lg-buddy status          # safe: query only
bin/lg-buddy on              # wake + select input
bin/lg-buddy sync-input      # re-select input (safe if the TV is already on)
bin/lg-buddy off             # power off
```

> Watch for the WebOS **1008 throttle** when testing `on`/`off` — rapid reconnects trip it and it doesn't self-clear, so space out repeated runs. Keep `PAIRING_DB` on a persistent path so a lost token doesn't force the TV to re-pair.

## Documentation

Keep documentation current as part of the change, not as a follow-up — update the README (and the `lg-buddy.env.example` knobs) in the same PR whenever behavior or config changes.

## Before you open a PR

- Lint with `shellcheck` and smoke-test the subcommand(s) your change affects against a real TV.
- For systemd unit or sleep-hook changes, run `sudo systemctl daemon-reload` and invoke the unit/hook manually (e.g. `sudo /lib/systemd/system-sleep/lg-buddy pre`) to confirm it's recognized and fires.
- For unlock-listener changes, verify it fires on unlock in a running graphical session.
- Never commit real credentials or the TV's IP/MAC — `*.env` is gitignored (only `*.env.example` is tracked).

## Branching & commits

- Branch off `main`; never commit directly to `main`.
- Use [Conventional Commits](https://www.conventionalcommits.org/) prefixes (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`, …).
- Sign your commits where possible (`git commit -S`).
- Keep each PR focused; delete dead code rather than commenting it out.

## Pull requests

- Open the PR against `main`.
- Every PR runs an automated code review — the only CI gate here. Resolve **all** review threads before the PR is merged.
- A PR can be merged once the review is clean and all review threads are resolved.

## Releases

Releases are opt-in. Add one of `semver:patch`, `semver:minor`, or `semver:major` to the PR to cut a release on merge; with no label, merging does not release. A release publishes an immutable `vX.Y.Z` tag and attaches the install tarball (`lg-buddy-<version>.tar.gz` + a `.sha256` checksum) bundling `bin/`, `systemd/`, `lg-buddy.env.example`, and the README.
