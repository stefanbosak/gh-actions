# Simple MS Teams Notifier

A GitHub Action to send simple notifications to Microsoft Teams via an Incoming Webhook.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `webhook-url` | ✅ | — | Microsoft Teams incoming webhook URL |
| `title` | ✅ | — | Notification title |
| `message` | ❌ | Auto-generated | Notification body text. Defaults to workflow name, branch, and actor. |
| `status` | ❌ | `info` | Visual status: `info` ℹ️, `success` ✅ , `failure` ❌ , `warning` ⚠️ |

---

## Outputs

| Output | Description |
|--------|-------------|
| `http-status` | HTTP status code returned by the Teams webhook |
| `success` | `true` if the notification was delivered, `false` otherwise |

## Usage examples

### Basic

```yaml
- uses: stefanbosak/ms-teams-simple-notifier@main
  with:
    webhook-url: ${{ secrets.TEAMS_WEBHOOK_URL }}
    title: 'Deployment complete'
```

### With all inputs

```yaml
- uses: stefanbosak/ms-teams-simple-notifier@main
  with:
    webhook-url: ${{ secrets.TEAMS_WEBHOOK_URL }}
    title: 'Production deploy'
    message: 'Version 1.2.3 was deployed successfully.'
    status: 'success'
```

### Notify on failure

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: ./deploy.sh

      - name: Notify Teams on failure
        if: failure()
        uses: stefanbosak/ms-teams-simple-notifier@main
        with:
          webhook-url: ${{ secrets.TEAMS_WEBHOOK_URL }}
          title: 'Deploy failed'
          status: 'failure'
```

### Build status notification

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: ./build.sh

      - name: 'Send notification'
        if: always()
        uses: stefanbosak/ms-teams-simple-notifier@main
        with:
          webhook-url: ${{ secrets.TEAMS_WEBHOOK_URL }}
          title: ${{ job.status == 'success' && 'Build Successful' || 'Build Failed' }}
          message: ${{ job.status == 'success' && format('{0} completed successfully', github.repository) || format('{0} failed', github.repository) }}
          status: ${{ job.status }}
```

### Use outputs

```yaml
- name: Notify Teams
  id: notify
  uses: stefanbosak/ms-teams-simple-notifier@main
  with:
    webhook-url: ${{ secrets.TEAMS_WEBHOOK_URL }}
    title: 'Build finished'
    status: 'success'

- name: Check result
  run: echo "Sent=${{ steps.notify.outputs.success }}, HTTP=${{ steps.notify.outputs.http-status }}"
```

## Setting up the webhook

1. In Microsoft Teams, go to the target channel → **⋯ More options** → **Connectors** (or **Workflows**).
2. Add an **Incoming Webhook**, give it a name, and copy the generated URL.
3. Store the URL as a secret:
   - **Repository secret**: repository **Settings → Secrets and variables → Actions → New repository secret** named `TEAMS_WEBHOOK_URL`.
   - **Organization secret** (share across multiple repos): organization **Settings → Secrets and variables → Actions → New organization secret** named `TEAMS_WEBHOOK_URL`, then set **Repository access** to the repos that should use it.

## Status styles

| Value | Icon | Card colour |
|-------|------|-------------|
| `success` | ✅ | Green |
| `failure` | ❌ | Red |
| `warning` | ⚠️ | Yellow |
| `info` | ℹ️ | Blue (default) |
