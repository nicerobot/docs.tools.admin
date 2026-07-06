---
title: tools.admin
---

**GitHub administration automation, shipped as both a Go CLI (`radm`) and a Docker-based GitHub Action.** It snapshots live repository settings into per-repo override files, opens the resulting change as a pull request, and prunes old GitHub Actions workflow runs.

- **Source:** [nicerobot/tools.admin](https://github.com/nicerobot/tools.admin)

`radm` is one binary with three subcommands — `snapshot`, `create-pr`, and `cleanup-runs`. Each renders a structured JSON result on stdout. Configuration comes from flags plus the standard GitHub Actions environment: `GH_TOKEN` (API token), `GITHUB_API_URL` (API base, defaulting to the public API when unset), and `GITHUB_REPOSITORY` (used to auto-detect the current repo).

## Install

As a CLI:

```sh
go install github.com/nicerobot/tools.admin/cmd/radm@latest
```

Or pull the released image published by the pipeline:

```sh
docker pull ghcr.io/nicerobot/tools.admin:v3
```

## Usage

### CLI

```sh
export GH_TOKEN=<a-github-token>

# Snapshot every repo under an owner into .github/repos/<name>.yml overrides
radm snapshot --owner nicerobot --settings-path .github

# Commit the snapshot onto a branch and open a PR
radm create-pr --settings-path .github --branch settings-sync/snapshot --base main

# Delete completed workflow runs older than 30 days, keeping the newest 5 per workflow
radm cleanup-runs --owner nicerobot --days 30 --keep 5 --dry-run
```

Root flags `--log-level` (`debug`, `info`, `warn`, `error`) and `--log-format` (`text`, `json`) control logging on stderr; `--version` prints the build version.

### GitHub Action

The repo root is a Docker action. Select a command with the `command` input:

```yaml
- uses: nicerobot/tools.admin@v3
  with:
    command: cleanup-runs
    cleanup-days: "30"
    cleanup-keep: "5"
    cleanup-dry-run: "false"
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Action inputs map onto the CLI flags per command:

- `command` (required) — `snapshot`, `create-pr`, or `cleanup-runs`.
- `owner` — GitHub user or organization. Required for `snapshot`; for `cleanup-runs` it auto-detects the current repo from `GITHUB_REPOSITORY` when omitted.
- `settings-path` — settings directory (default `.github`).
- `branch` / `base` — for `create-pr` (defaults `settings-sync/snapshot` / `main`).
- `cleanup-repo` — limit `cleanup-runs` to a single repo (omit for all repos under the owner).
- `cleanup-days` / `cleanup-keep` / `cleanup-dry-run` — retention window, per-workflow keep count, and dry-run toggle (defaults `30` / `5` / `false`).

## Commands

### `snapshot`

For the given `--owner`, lists every repository, diffs each against the org or account defaults in `<settings-path>/settings.yml`, and writes a `<settings-path>/repos/<name>.yml` override for it. Override files for repositories that no longer exist are removed — but only after confirming each is truly gone, so a token that cannot see every repo never deletes a file it merely failed to list.

### `create-pr`

Commits the snapshot under `<settings-path>/repos` onto `--branch` and opens a pull request into `--base`. It configures the `github-actions[bot]` identity, force-creates the branch, and stages the repos directory; if nothing is staged the command is a no-op. Otherwise it commits, force-pushes, and opens a PR — unless one is already open for the branch.

### `cleanup-runs`

For the target owner (or the current repository from `GITHUB_REPOSITORY` when `--owner` is omitted) and `--repo` (or every repo under the owner when omitted), lists completed runs older than `--days`, groups them by workflow, keeps the newest `--keep` per workflow, and deletes the rest. With `--dry-run`, nothing is deleted and the result reports what would be.

## Design

`radm` is built on [urfave/cli/v3](https://github.com/urfave/cli). Each subcommand is a thin CLI wrapper over an injectable domain package: OS-backed collaborators (the GitHub client from `GH_TOKEN`/`GITHUB_API_URL`, git command execution, the clock, environment lookup) are wired at the edge so the core logic is exercised without real network or filesystem access. The published Docker image runs the same binary through `entrypoint.sh`, which translates the `INPUT_*` action inputs into `radm` invocations.
