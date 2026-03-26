# GitHub Actions Collection

A collection of reusable GitHub composite actions for common CI/CD tasks.

| Action | Description |
|--------|-------------|
| [cleanup-dockerhub-old-digests](#cleanup-dockerhub-old-digests) | Delete obsolete image manifests from DockerHub after a successful build |
| [get-repo-description](#get-repo-description) | Fetch a GitHub repository's description via the API |
| [ms-teams-simple-notifier](#ms-teams-simple-notifier) | Send formatted status notifications to Microsoft Teams |

---

## cleanup-dockerhub-old-digests

**Path:** [`cleanup-dockerhub-old-digests/`](./cleanup-dockerhub-old-digests/)

Locates the previous successful workflow run for the same branch, downloads its `digests-all` artifact (a plain-text list of manifest digests), and bulk-deletes those digests from the DockerHub Container Registry using the Docker Registry v2 API. Keeps your registry lean by removing image layers that are no longer needed after a new build.

### Quick usage

```yaml
- uses: stefanbosak/cleanup-dockerhub-old-digests@main
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    repository: ${{ github.repository }}
    run-id: ${{ github.run_id }}
    ref-name: ${{ github.ref_name }}
    dh-user: ${{ secrets.DOCKERHUB_USERNAME }}
    dh-token: ${{ secrets.DOCKERHUB_TOKEN }}
    repository-name: my-image
```

### Inputs

| Name | Required | Description |
|------|----------|-------------|
| `github-token` | ✅ | GitHub token (actions:read + artifact download) |
| `repository` | ✅ | `owner/repo` |
| `run-id` | ✅ | Current run ID (excluded from previous-run search) |
| `ref-name` | ✅ | Branch/tag to scope the search |
| `dh-user` | ✅ | DockerHub username |
| `dh-token` | ✅ | DockerHub access token (delete scope) |
| `repository-name` | ✅ | Image name on DockerHub (no owner prefix) |

### Outputs

| Name | Description |
|------|-------------|
| `prev-run-id` | ID of the previous successful run |

📖 [Full documentation](./cleanup-dockerhub-old-digests/README.md)

**References:** [Docker Registry v2 API](https://distribution.github.io/distribution/spec/api/#deleting-an-image) · [GitHub Actions API — workflow runs](https://docs.github.com/en/rest/actions/workflow-runs) · [actions/download-artifact](https://github.com/actions/download-artifact)

---

## get-repo-description

**Path:** [`get-repo-description/`](./get-repo-description/)

A minimal action that calls the GitHub REST API to retrieve a repository's description and exposes it as a step output. Useful when you need the description string dynamically (e.g., to populate release notes, Slack messages, or documentation artifacts).

### Quick usage

```yaml
- uses: stefanbosak/get-repo-description@main
  id: desc
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    # repository: owner/name  # optional, defaults to current repo

- run: echo "${{ steps.desc.outputs.description }}"
```

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `github-token` | ✅ | — | GitHub token for API authentication |
| `repository` | ❌ | `${{ github.repository }}` | Repository in `owner/name` format |

### Outputs

| Name | Description |
|------|-------------|
| `description` | The repository description string |

📖 [Full documentation](./get-repo-description/README.md)

**References:** [GitHub REST API — Get a repository](https://docs.github.com/en/rest/repos/repos#get-a-repository) · [jq manual](https://jqlang.github.io/jq/manual/)

---

## ms-teams-simple-notifier

**Path:** [`ms-teams-simple-notifier/`](./ms-teams-simple-notifier/)

Sends a richly formatted Adaptive Card notification to a Microsoft Teams channel via an Incoming Webhook. Supports four visual status styles (`success` ✅, `failure` ❌, `warning` ⚠️, `info` ℹ️) and automatically includes repository, branch, actor, and commit context. The card also provides one-click links to the workflow run and the commit.

### Quick usage

```yaml
- uses: stefanbosak/ms-teams-simple-notifier@main
  with:
    webhook-url: ${{ secrets.TEAMS_WEBHOOK_URL }}
    title: 'Deployment complete'
    status: 'success'
```

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `webhook-url` | ✅ | — | Teams incoming webhook URL |
| `title` | ✅ | — | Card title |
| `message` | ❌ | Auto-generated | Card body text |
| `status` | ❌ | `info` | `success` / `failure` / `warning` / `info` |

### Outputs

| Name | Description |
|------|-------------|
| `http-status` | HTTP status code from the webhook call |
| `success` | `true` if delivered successfully |

📖 [Full documentation](./ms-teams-simple-notifier/README.md)

**References:** [Microsoft Teams Incoming Webhooks](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook) · [Adaptive Cards schema](https://adaptivecards.io/explorer/) · [Microsoft Teams Workflows app](https://support.microsoft.com/en-us/office/create-incoming-webhooks-with-workflows-for-microsoft-teams-8ae491c7-0394-4861-ba59-055e33f75498)
