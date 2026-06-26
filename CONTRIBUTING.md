# Contributing to lg-buddy

`lg-buddy` keeps an LG WebOS TV in sync with a Linux host's power and session state — a pure Bash CLI plus systemd units, no compiled artifacts. All changes go through the workflow below.

## Prerequisites

- `bash`
- `python3` with [`bscpylgtv`](https://github.com/chros73/bscpylgtv) — the WebOS control library the scripts shell out to
- A reachable LG WebOS TV (with a saved pairing DB) to exercise behavior end-to-end
- `shellcheck` (optional, recommended) — to lint the scripts locally

## Build, test & lint

This is pure Bash — there is no build step, test suite, or lint job in CI; the automated PR review is the quality gate. Before pushing, lint the scripts and smoke-test against a real TV:

```bash
# Lint the shell scripts locally (recommended; not run in CI)
shellcheck bin/lg-buddy bin/lg-buddy-unlock-listen

# Smoke-test the entry point against your configured TV
LG_BUDDY_ENV=/etc/lg-buddy/lg-buddy.env bin/lg-buddy status
```

## Documentation

Keep documentation current as part of the change, not as a follow-up — update the README (and the `lg-buddy.env.example` knobs) in the same PR whenever behavior or config changes.

## Before you open a PR

- Lint with `shellcheck` and smoke-test the affected path against a real TV.
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
