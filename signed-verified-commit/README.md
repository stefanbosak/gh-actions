# Signed and Verified Commit and Push

A GitHub composite action that commits and pushes one or more files to a branch using the GitHub GraphQL `createCommitOnBranch` mutation. Because this mutation is backed by the GitHub API, every commit is automatically **signed and marked as verified** by `github-actions[bot]` — no GPG key setup required.

## How it works

1. **Resolve target branch** — determines the destination branch from the `branch` input, the PR head ref (`github.head_ref`), or the current `GITHUB_REF`, in that priority order.
2. **Read current HEAD OID** — fetches the latest commit SHA on the target branch via the GitHub REST API (`git.getRef`). This OID is required by the GraphQL mutation to prevent accidental overwrites.
3. **Build file additions** — reads every specified file from the runner filesystem and Base64-encodes its content. Supports an optional `repo/path:local/path` mapping syntax when the in-repo path differs from the local path.
4. **Create verified commit** — calls the `createCommitOnBranch` GraphQL mutation to atomically commit all additions and push to the branch. GitHub signs the commit on behalf of the app, so it shows the **Verified** badge in the UI.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `github-token` | ✅ | — | GitHub token (`secrets.GITHUB_TOKEN`) with `contents: write` permission |
| `commit-message` | ✅ | — | Commit headline message |
| `files` | ✅ | — | Newline-separated list of files to commit (see [File syntax](#file-syntax)) |
| `branch` | ❌ | `''` | Target branch. Defaults to the PR head branch or the branch that triggered the workflow |
| `repository` | ❌ | `${{ github.repository }}` | Target repository in `owner/repo` format |

### File syntax

Each line in `files` is one of:

```
path/to/file                # repo path == local runner path
repo/path:local/path        # different in-repo path and local path
```

## Outputs

| Output | Description |
|--------|-------------|
| `commit-sha` | SHA (OID) of the newly created commit |
| `commit-url` | Web URL of the newly created commit |

## Usage example

```yaml
jobs:
  update:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4

      - name: Generate output
        run: echo "built at $(date -u)" > output.txt

      - name: Commit and push (verified)
        id: verified-commit
        uses: YOUR_ORG/YOUR_REPO/.github/actions/signed-verified-commit@main
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: 'chore: automated update [skip ci]'
          files: |
            output.txt
            dist/bundle.js:build/output.js

      - name: Print commit info
        run: |
          echo "SHA: ${{ steps.verified-commit.outputs.commit-sha }}"
          echo "URL: ${{ steps.verified-commit.outputs.commit-url }}"
```

> See [`example-usage.yml`](./example-usage.yml) for a complete workflow with `workflow_dispatch` support and a custom target branch.

## References

- [GitHub GraphQL API — `createCommitOnBranch` mutation](https://docs.github.com/en/graphql/reference/mutations#createcommitonbranch)
- [GitHub Actions — `actions/github-script`](https://github.com/actions/github-script)
- [GitHub — Commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification)
