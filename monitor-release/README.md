# Monitor Release

A composite GitHub Action that monitors one or more repositories for new releases and fires a `repository_dispatch` event whenever a new release is detected on any of them.

> **Pre-release preference** — when the most recent pre-release is _newer_ than the latest stable release, the pre-release is treated as the current version.
>
> **Tag mode** — set `"usetag": true` on any entry to resolve the current version from git tags instead of GitHub Releases. Useful when a project pushes tags but does not maintain GitHub Releases.

---

## Usage

### Single repository

```yaml
- uses: your-org/monitor-release@main
  with:
    projects: '[{"owner":"octocat","repo":"hello-world"}]'
```

### Multiple repositories

```yaml
- uses: your-org/monitor-release@main
  with:
    projects: |
      [
        {"owner": "octocat",  "repo": "hello-world"},
        {"owner": "octocat",  "repo": "linguist"},
        {"owner": "torvalds", "repo": "linux"}
      ]
```

### Tag mode (skip GitHub Releases, resolve from git tags)

```yaml
- uses: your-org/monitor-release@main
  with:
    projects: |
      [
        {"owner": "octocat", "repo": "hello-world"},
        {"owner": "acme",    "repo": "legacy-lib", "usetag": true}
      ]
```

### Object / map form (one repo per owner)

```yaml
- uses: your-org/monitor-release@main
  with:
    projects: '{"octocat":"hello-world","torvalds":"linux"}'
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
    permissions:
      actions: write   # read/write cache + delete old entries
      contents: read   # list releases
    steps:
      - uses: actions/checkout@v4

      - uses: your-org/monitor-release@main
        with:
          projects: |
            [
              {"owner": "octocat", "repo": "hello-world"},
              {"owner": "octocat", "repo": "linguist"}
            ]
```

---

## Inputs

| Name | Required | Description |
|---|---|---|
| `projects` | ✅ | JSON **array** of `{owner, repo, usetag?}` objects **or** a JSON **object** mapping `owner → repo`. See formats below. |

### Input formats

Both formats are supported. Per-project options (such as `usetag`) are only available in the **array** format.

**Array** — preferred when you have multiple repos under different owners, or need per-project options:

```json
[
  {"owner": "acme", "repo": "widget"},
  {"owner": "acme", "repo": "gadget"},
  {"owner": "other-org", "repo": "tool", "usetag": true}
]
```

**Object / map** — convenient shorthand when each owner appears once; always uses release mode:

```json
{"acme": "widget", "other-org": "tool"}
```

### Per-project options (array format only)

| Field | Type | Default | Description |
|---|---|---|---|
| `owner` | string | — | Repository owner (user or org) |
| `repo` | string | — | Repository name |
| `usetag` | boolean | `false` | When `true`, resolves the current version from git tags (`listTags`) instead of GitHub Releases. Picks the most recent tag. Useful when a project does not keep GitHub Releases up-to-date. |

---

## Outputs

This action produces no declared outputs. Side effects are described below.

---

## How it works

```
┌──────────────────────────────────────────────────┐
│ 1. Restore cache                                 │
│    – restore-keys: last-known-                   │
│    → reads .last-known-tags.json (if any)        │
│      { "owner/repo": "vX.Y.Z", … }              │
└───────────────────┬──────────────────────────────┘
                    │ cached tag map
┌───────────────────▼──────────────────────────────┐
│ 2. For each (owner, repo, usetag?) pair:         │
│                                                  │
│    usetag=true          usetag=false (default)   │
│    ────────────         ─────────────────────    │
│    listTags API         listReleases API          │
│    pick tags[0]         pick pre-release if       │
│    type = "tag"          newer than stable        │
│    meta = {commit}      type = "release"|"pre…"  │
│                         meta = {published_at,     │
│                                 html_url}         │
│    ──────────────────────────────────────────    │
│    b. Compare selected tag with cached tag        │
│       if different → hasNewRelease = true         │
│    c. Push {owner,repo,tag,type,...meta}          │
│       to changes[] (every fetched project)        │
│    d. Update tag in the working map               │
│    (API errors for one repo are skipped;          │
│     remaining repos continue normally)            │
│    ──────────────────────────────────────────    │
│    After loop — if hasNewRelease:                 │
│    → github.repos.createDispatchEvent  (once)     │
│         event_type:    new-release                │
│         client_payload: { changes }  ← all       │
└───────────────────┬──────────────────────────────┘
                    │ updated tag map written to disk
┌───────────────────▼──────────────────────────────┐
│ 3. Save updated map to cache                     │
│    key: last-known-{run_id}  (unique/run)        │
│    then delete all but the newest entry          │
└──────────────────────────────────────────────────┘
```

1. **Restore cache** — uses `actions/cache` restore-keys to retrieve the previously recorded tag map from `.last-known-tags.json`. On the first run the file does not exist and all repos are treated as new.
2. **Fetch, compare, and dispatch** — for each project entry, the resolution strategy depends on `usetag`:
   - **Release mode** (`usetag` absent or `false`): fetches up to 10 recent releases via `listReleases`, selects the best tag (pre-release preferred when newer than the latest stable release). Metadata included: `published_at`, `html_url`.
   - **Tag mode** (`usetag: true`): fetches up to 10 tags via `listTags` (newest first) and picks the first. Metadata included: `commit` (SHA). Useful when a project publishes git tags but does not maintain GitHub Releases.

   In both modes, the selected tag is compared against the cached value. If it differs, `hasNewRelease` is set to `true`. Either way the entry `{owner, repo, tag, type, …meta}` is appended to the `changes` array. After all pairs are processed, if `hasNewRelease` is `true` a single [`repository_dispatch`](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event) event is fired; the `changes` payload always contains **all** successfully fetched projects — not just the ones that changed — so downstream workflows have a complete snapshot of every monitored repo. A failed API call for one pair is logged and skipped; the remaining pairs are processed normally.
3. **Update cache** — writes the full updated tag map with a run-unique cache key, then purges all older `last-known-*` cache entries to keep storage tidy.

---

## Dispatch event

When one or more monitored repositories release simultaneously, the action fires **one** dispatch event on the calling repository carrying all changes:

| Field | Value |
|---|---|
| `event_type` | `new-release` |
| `client_payload.changes` | Array — **one entry per successfully fetched repository** (all monitored projects, not only those that changed) |
| `changes[N].owner` | Repository owner |
| `changes[N].repo` | Repository name |
| `changes[N].tag` | Current tag (e.g. `v1.2.3`) |
| `changes[N].type` | How the tag was resolved: `"release"`, `"prerelease"`, or `"tag"` |
| `changes[N].commit` | _(tag mode only)_ Commit SHA the tag points to |
| `changes[N].published_at` | _(release mode only)_ ISO 8601 publish timestamp of the selected release |
| `changes[N].html_url` | _(release mode only)_ URL of the GitHub Release page |

Exactly one event is fired per run, regardless of how many monitored repos released simultaneously. The event is only fired when at least one repo has a new release; if nothing changed, no event is fired.

A downstream workflow can listen for this event:

```yaml
on:
  repository_dispatch:
    types: [new-release]

jobs:
  handle:
    runs-on: ubuntu-latest
    steps:
      - name: Print all detected changes
        run: echo '${{ toJson(github.event.client_payload.changes) }}'

      - name: Act on first change (single-project monitoring)
        run: |
          echo "New release: ${{ github.event.client_payload.changes[0].owner }}/${{ github.event.client_payload.changes[0].repo }}"
          echo "Tag:  ${{ github.event.client_payload.changes[0].tag }}"
          echo "Type: ${{ github.event.client_payload.changes[0].type }}"
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
| Cache path | `.last-known-tags.json` |
| Cache format | JSON object — `{ "owner/repo": "vX.Y.Z", … }` |
| Save key pattern | `last-known-{run_id}` |
| Restore key prefix | `last-known-` |
| Retention policy | Only the single most-recent entry is kept; all others are deleted each run |

> Because a new unique key is used every run, GitHub Cache never rejects the save with an "entry already exists" error.
>
> Adding a new repo to `projects` is safe — its key simply won't exist in the cached map yet, so the first run treats it as a new release.
>
> Removing a repo from `projects` leaves a stale key in the JSON map, but it is harmlessly ignored on every subsequent run.

---

## Requirements

| Dependency | Notes |
|---|---|
| `actions/github-script@main` | Used for GitHub API calls and cache cleanup |
| `actions/cache/restore@main` | Restores the last-known tag map |
| `actions/cache/save@main` | Saves the updated tag map |
