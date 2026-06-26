# Monitor Release

A composite GitHub Action that monitors one or more repositories for new releases and fires a `repository_dispatch` event whenever a new release is detected on any of them.

> **Latest-release resolution** — for repositories without a `tagprefix`, the action uses GitHub's `GET /releases/latest` endpoint to identify the current stable release. This correctly surfaces newer major versions (e.g. Helm v4 over v3) the moment GitHub marks them *latest*, without relying on date-sorting heuristics.
>
> **Pre-release preference** — when the most recent pre-release is _newer_ than the latest stable release, the pre-release is treated as the current version.
>
> **Tag mode** — set `"usetag": true` on any entry to resolve the current version from git tags instead of GitHub Releases. Useful when a project pushes tags but does not maintain GitHub Releases.
>
> **Monorepo / subproject mode** — set `"tagprefix": "kyaml/"` to track only releases whose tag starts with that prefix. Each unique `owner/repo/tagprefix` combination is tracked independently, so multiple subprojects within a single repository can all be monitored simultaneously.

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

### Monorepo subprojects (e.g. kustomize)

```yaml
- uses: your-org/monitor-release@main
  with:
    projects: |
      [
        {"owner": "kubernetes-sigs", "repo": "kustomize", "tagprefix": "kustomize/"},
        {"owner": "kubernetes-sigs", "repo": "kustomize", "tagprefix": "kyaml/"}
      ]
```

Each entry is tracked independently: a new `kyaml/` release fires the same dispatch event as any other new release, and its `tagprefix` field in the payload tells downstream workflows which subproject changed.

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
| `projects` | ✅ | JSON **array** of `{owner, repo, usetag?, tagprefix?}` objects **or** a JSON **object** mapping `owner → repo`. See formats below. |

### Input formats

Both formats are supported. Per-project options (such as `usetag` and `tagprefix`) are only available in the **array** format.

**Array** — preferred when you have multiple repos under different owners, or need per-project options:

```json
[
  {"owner": "acme", "repo": "widget"},
  {"owner": "acme", "repo": "gadget"},
  {"owner": "other-org", "repo": "tool", "usetag": true},
  {"owner": "kubernetes-sigs", "repo": "kustomize", "tagprefix": "kyaml/"}
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
| `usetag` | boolean | `false` | When `true`, resolves the current version from git tags (`listTags`) instead of GitHub Releases. Picks the most recent matching tag. Useful when a project does not keep GitHub Releases up-to-date. |
| `tagprefix` | string | `""` | When set, only releases/tags whose name starts with this string are considered. Enables independent tracking of multiple subprojects within a single monorepo (e.g. `"kyaml/"` in `kubernetes-sigs/kustomize`). The cache key becomes `owner/repo#tagprefix`, so each prefix is tracked separately. |

---

## Outputs

This action produces no declared outputs. Side effects are described below.

---

## How it works

```
┌──────────────────────────────────────────────────────┐
│ 1. Restore cache                                     │
│    – restore-keys: last-known-                       │
│    → reads .last-known-tags.json (if any)            │
│      { "owner/repo": "vX.Y.Z",                       │
│        "owner/repo#tagprefix": "prefix/vX.Y.Z", … } │
└───────────────────┬──────────────────────────────────┘
                    │ cached tag map
┌───────────────────▼──────────────────────────────────┐
│ 2. For each (owner, repo, usetag?, tagprefix?) pair: │
│    cache key = owner/repo  or  owner/repo#tagprefix  │
│                                                      │
│    usetag=true            usetag=false (default)     │
│    ────────────           ─────────────────────      │
│    listTags API           no tagprefix:               │
│    paginate (≤10×100)       getLatestRelease API      │
│    filter by tagprefix    with tagprefix:             │
│    pick first match         listReleases (≤10×100)   │
│    type = "tag"           pick newest pre-release     │
│    meta = {commit}         and "latest"; newer wins   │
│                           type = "release"|"pre…"    │
│                           meta = {published_at,       │
│                                   html_url}           │
│    ────────────────────────────────────────────      │
│    b. Compare selected tag with cached tag            │
│       if different → hasNewRelease = true             │
│    c. Push {owner,repo,tag,type,tagprefix,...meta}    │
│       to changes[] (every fetched project)            │
│    d. Update tag in the working map                   │
│    (API errors for one repo are skipped;              │
│     remaining repos continue normally)                │
│    ────────────────────────────────────────────      │
│    After loop — if hasNewRelease:                     │
│    → github.repos.createDispatchEvent  (once)         │
│         event_type:    new-release                    │
│         client_payload: { changes }  ← all           │
└───────────────────┬──────────────────────────────────┘
                    │ updated tag map written to disk
┌───────────────────▼──────────────────────────────────┐
│ 3. Save updated map to cache                         │
│    key: last-known-{run_id}  (unique/run)            │
│    then delete all but the newest entry              │
└──────────────────────────────────────────────────────┘
```

1. **Restore cache** — uses `actions/cache` restore-keys to retrieve the previously recorded tag map from `.last-known-tags.json`. On the first run the file does not exist and all repos are treated as new.
2. **Fetch, compare, and dispatch** — for each project entry, the resolution strategy depends on `usetag`:
   - **Release mode** (`usetag` absent or `false`): resolution varies by whether `tagprefix` is set.
     - **No `tagprefix`**: calls `GET /repos/{owner}/{repo}/releases/latest` (GitHub's own *latest* designation — most-recent non-prerelease, non-draft release by `created_at`) for the stable candidate, then paginates `listReleases` to find the most-recent pre-release. Using the dedicated endpoint means a repository that has promoted a new major version to *latest* (e.g. Helm v4 over v3) is detected immediately.
     - **With `tagprefix`**: paginates up to 10 pages of 100 releases via `listReleases`, filtering each page to releases whose `tag_name` starts with the prefix; pagination stops as soon as both the most-recent pre-release and stable release matching the prefix are found. (`getLatestRelease` is not prefix-aware, so pagination is required here.)

     In both cases, selects whichever candidate is newer by `published_at` (pre-release wins when newer than stable). Metadata included: `published_at`, `html_url`.
   - **Tag mode** (`usetag: true`): paginates up to 10 pages of 100 tags each via `listTags` (newest first). When `tagprefix` is set, picks the first tag whose name starts with the prefix; otherwise picks the very first tag. Metadata included: `commit` (SHA). Useful when a project publishes git tags but does not maintain GitHub Releases.

   Each entry is identified by a **cache key** of `owner/repo` (no prefix) or `owner/repo#tagprefix` (with prefix), so multiple subprojects of the same repository are tracked independently. The selected tag is compared against the cached value for that key. If it differs, `hasNewRelease` is set to `true`. Either way the entry `{owner, repo, tag, type, tagprefix, …meta}` is appended to the `changes` array. After all pairs are processed, if `hasNewRelease` is `true` a single [`repository_dispatch`](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event) event is fired; the `changes` payload always contains **all** successfully fetched projects — not just the ones that changed — so downstream workflows have a complete snapshot of every monitored repo. A failed API call for one pair is logged and skipped; the remaining pairs are processed normally.
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
| `changes[N].tag` | Current tag (e.g. `v1.2.3` or `kyaml/v0.21.1`) |
| `changes[N].type` | How the tag was resolved: `"release"`, `"prerelease"`, or `"tag"` |
| `changes[N].tagprefix` | The `tagprefix` filter that was applied, or `null` when not used |
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
| Cache format | JSON object — `{ "owner/repo": "vX.Y.Z", "owner/repo#tagprefix": "prefix/vX.Y.Z", … }` |
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
