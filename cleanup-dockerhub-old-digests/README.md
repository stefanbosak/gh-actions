# Cleanup DockerHub Old Digests

A GitHub composite action that identifies the previous successful workflow run, downloads its stored image-digest artifact, and bulk-deletes those manifest digests from the DockerHub Container Registry. This keeps your registry clean by removing image layers that are no longer referenced by the latest build.

## How it works

1. **Find previous run** — uses [`actions/github-script`](https://github.com/actions/github-script) to call the GitHub Actions REST API and locate the most recent *successful* run of the same workflow on the same branch (excluding the current run).
2. **Download digest list** — fetches the `digests-all` artifact that was uploaded by that previous run (expected path inside the artifact: `/tmp/digests-all.txt`, one digest per line).
3. **Delete manifests** — authenticates with the DockerHub Registry v2 API and issues a `DELETE /v2/<repo>/manifests/<digest>` request for every digest in the file.

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| `github-token` | ✅ | GitHub token (needs `actions:read` for the API calls and artifact download) |
| `repository` | ✅ | Repository in `owner/repo` format |
| `run-id` | ✅ | Current workflow run ID — used to exclude it when searching for the previous run |
| `ref-name` | ✅ | Branch or tag name used to scope the previous-run search |
| `dh-user` | ✅ | DockerHub username |
| `dh-token` | ✅ | DockerHub access token (needs `delete` scope) |
| `repository-name` | ✅ | Image / repository name on DockerHub (without the `owner/` prefix) |

---

## Outputs

| Output | Description |
|--------|-------------|
| `prev-run-id` | The ID of the previous successful workflow run, or empty/`null` if none was found |

## Usage example

```yaml
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      # ... build steps that produce a digests-all artifact ...

      - name: Cleanup old DockerHub digests
        uses: stefanbosak/cleanup-dockerhub-old-digests@main
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          repository: ${{ github.repository }}
          run-id: ${{ github.run_id }}
          ref-name: ${{ github.ref_name }}
          dh-user: ${{ secrets.DOCKERHUB_USERNAME }}
          dh-token: ${{ secrets.DOCKERHUB_TOKEN }}
          repository-name: my-image
```

### Expected artifact format

The action looks for an artifact named **`digests-all`** in the previous run. The artifact must contain a plain-text file `/tmp/digests-all.txt` with one manifest digest per line, e.g.:

```
sha256:abc123...
sha256:def456...
```

> If the file is missing (no previous run, or the artifact was not uploaded), the cleanup step is skipped gracefully.

## References

- [GitHub Actions — Download Artifact (`actions/download-artifact`)](https://github.com/actions/download-artifact)
- [GitHub Actions — `actions/github-script`](https://github.com/actions/github-script)
- [GitHub REST API — List workflow runs](https://docs.github.com/en/rest/actions/workflow-runs#list-workflow-runs-for-a-workflow)
- [Docker Registry HTTP API v2 — Delete Manifest](https://distribution.github.io/distribution/spec/api/#deleting-an-image)
- [DockerHub Access Tokens](https://docs.docker.com/security/for-developers/access-tokens/)
