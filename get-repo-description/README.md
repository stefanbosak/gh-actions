# Get Repository Description

A lightweight GitHub composite action that fetches the description of any GitHub repository using the GitHub REST API and exposes it as a step output for use in subsequent workflow steps.

## How it works

The action calls `GET /repos/{owner}/{repo}` via the GitHub API and extracts the `.description` field with `jq`, then writes it to `$GITHUB_OUTPUT` so downstream steps can reference it.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `github-token` | ✅ | — | GitHub token for API authentication (needs `repo` or `public_repo` read access) |
| `repository` | ❌ | `${{ github.repository }}` | Repository to query, in `owner/name` format. Defaults to the current repository. |

---

## Outputs

| Output | Description |
|--------|-------------|
| `description` | The repository description string (empty string if not set) |

## Usage examples

### Current repository (default)

```yaml
- name: Get repo description
  id: desc
  uses: stefanbosak/get-repo-description@main
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}

- name: Print description
  run: echo "${{ steps.desc.outputs.description }}"
```

### Another repository

```yaml
- name: Get description of another repo
  id: desc
  uses: stefanbosak/get-repo-description@main
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    repository: octocat/Hello-World

- name: Use the description
  run: echo "Description is: ${{ steps.desc.outputs.description }}"
```

### Conditional step based on description

```yaml
- name: Get description
  id: desc
  uses: stefanbosak/get-repo-description@main
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}

- name: Only run if description is set
  if: steps.desc.outputs.description != ''
  run: echo "Repo has a description!"
```

## References

- [GitHub REST API — Get a repository](https://docs.github.com/en/rest/repos/repos#get-a-repository)
- [GitHub Actions — Defining outputs for jobs](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/passing-information-between-jobs)
- [`jq` manual](https://jqlang.github.io/jq/manual/)
