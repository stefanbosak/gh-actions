# Monitor Release

A composite GitHub Action that monitors a repository for new releases and fires a `repository_dispatch` event whenever a new release is detected.

> **Pre-release preference** — when the most recent pre-release is _newer_ than the latest stable release, the pre-release is treated as the current version.

---

## Usage

```yaml
- uses: ./ # or your-org/monitor-release@main
  with:
    project_owner: octocat
    project_repo: hello-world
```

### Minimal workflow example

```yaml
name: Check for new releases

on:
  schedule:
    - cron: '0 * * * *'   # every hour
  workflow_dispatch:

jobs:
  monitor:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: your-org/monitor-release@main
        with:
          project_owner: octocat
          project_repo: hello-world
```

---

## Inputs

| Name | Required | Description |
|---|---|---|
| `project_owner` | ✅ | Owner of the repository to monitor (user or org name) |
| `project_repo` | ✅ | Name of the repository to monitor |

---

## Outputs

This action produces no declared outputs. Side effects are described below.

---

## How it works

```
┌──────────────────────────────────────────────┐
│ 1. Fetch releases                            │
│    – latest stable release                  │
│    – latest pre-release                     │
│    → pick the one with the newer date        │
└───────────────────┬──────────────────────────┘
                    │ selected tag
┌───────────────────▼──────────────────────────┐
│ 2. Restore cache                             │
│    – restore-keys: last-known-               │
│    → reads .last-known-tag (if any)          │
└───────────────────┬──────────────────────────┘
                    │ cached tag
┌───────────────────▼──────────────────────────┐
│ 3. Compare tags                              │
│    if selected ≠ cached (and not empty)      │
│    → github.repos.createDispatchEvent        │
│       event_type: new-release                │
│       client_payload: { tag }                │
└───────────────────┬──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│ 4. Save new tag to cache                     │
│    key: last-known-{run_id}  (unique/run)    │
│    then delete all but the newest entry      │
└──────────────────────────────────────────────┘
```

1. **Fetch releases** — queries up to the 10 most recent releases of the target repository and picks the most appropriate tag (pre-release preferred when newer).
2. **Restore cache** — uses `actions/cache` restore-keys to retrieve the previously recorded tag from `.last-known-tag`.
3. **Compare and dispatch** — if the selected tag differs from the cached tag, a [`repository_dispatch`](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event) event is fired on the **calling** repository so downstream workflows can react.
4. **Update cache** — writes the selected tag with a run-unique cache key, then purges all older `last-known-*` cache entries to keep storage tidy.

---

## Dispatch event

When a new release is detected, the action fires a dispatch event on the calling repository:

| Field | Value |
|---|---|
| `event_type` | `new-release` |
| `client_payload.tag` | The newly detected tag (e.g. `v1.2.3`) |

A downstream workflow can listen for this event:

```yaml
on:
  repository_dispatch:
    types: [new-release]

jobs:
  handle:
    runs-on: ubuntu-latest
    steps:
      - run: echo "New release ${{ github.event.client_payload.tag }}"
```

---

## Permissions

The workflow that uses this action must have the following permissions:

```yaml
permissions:
  actions: write   # read/write cache + delete old entries
  contents: read   # list releases
```

If `repository_dispatch` must be sent to a **different** repository, a Personal Access Token (PAT) or GitHub App token with `repo` scope must be supplied via the `GITHUB_TOKEN` environment variable.

---

## Cache behaviour

| Detail | Value |
|---|---|
| Cache path | `.last-known-tag` |
| Save key pattern | `last-known-{run_id}` |
| Restore key prefix | `last-known-` |
| Retention policy | Only the single most-recent entry is kept; all others are deleted each run |

> Because a new unique key is used every run, GitHub Cache never rejects the save with an "entry already exists" error.

---

## Requirements

| Dependency | Notes |
|---|---|
| `actions/github-script@main` | Used for GitHub API calls |
| `actions/cache/restore@main` | Restores the last-known tag |
| `actions/cache/save@main` | Saves the current tag |
