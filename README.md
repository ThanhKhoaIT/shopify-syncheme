# shopify-syncheme

Keep a Shopify theme repository in sync with edits made directly in Shopify admin.

Marketing and merchandising teams change the storefront in the theme editor: section order,
copy, images, settings. Those edits live only on Shopify until someone pulls them. This
reusable GitHub Actions workflow pulls the theme on a schedule and opens (or updates) **one
pull request** with the changes, so a developer can review and merge them.

- **Read-only against Shopify.** It runs `shopify theme pull` only; it never pushes or publishes a theme.
- **One PR at a time.** Every run rebuilds a fixed branch (`sync/admin-edits`) from your base branch.
  An open PR is updated in place; after you merge, the next change opens a new one.
- **Merchant-owned files by default.** Only the JSON files the theme editor writes are pulled, so
  code you merged but have not published yet is not shown as "reverted" in the PR.
- **Monorepo friendly.** The theme can live in a subfolder (`theme-path`).

## How it works

```
schedule (e.g. every 30 min)
  └─ shopify theme pull --live --only templates/*.json ... --path <theme-path>
       └─ changes?  ── no ──> nothing happens
            └─ yes ──> commit to sync/admin-edits, open or update the PR "Sync theme updates"
```

## Installation

### 1. Create a Theme Access password

1. In Shopify admin, install the [Theme Access](https://apps.shopify.com/theme-access) app (by Shopify).
2. Open it and choose **Create password**, then enter a name (e.g. `github-sync`) and an email.
3. Open the email link and copy the password (`shptka_...`). It can read and write themes only.

Repeat for each store you want to sync.

### 2. Add the secret to your repository

Go to **Settings → Secrets and variables → Actions → New repository secret**:

| Name | Value |
| --- | --- |
| `SHOPIFY_THEME_ACCESS_PASSWORD` | the `shptka_...` password |

When several repositories share one store, use an organization secret instead.

### 3. Allow Actions to open pull requests

Go to **Settings → Actions → General → Workflow permissions** and tick
**Allow GitHub Actions to create and approve pull requests**.
For repositories in an organization, the same option must also be enabled at organization level.

### 4. Add the workflow

Create `.github/workflows/sync-admin-edits.yml` (see [`examples/sync-admin-edits.yml`](examples/sync-admin-edits.yml)):

```yaml
name: Sync admin edits

on:
  schedule:
    # Cron is UTC. Vietnam (ICT) is UTC+7.
    - cron: "5 1-13 * * 1-5"     # Mon–Fri, hourly 08:05–20:05 ICT
    - cron: "5 1-13/4 * * 0,6"   # Sat–Sun, 08:05 / 12:05 / 16:05 / 20:05 ICT
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    uses: ThanhKhoaIT/shopify-syncheme/.github/workflows/sync.yml@v1
    with:
      store: your-store            # your-store.myshopify.com
      theme-path: themes/main      # "." if the theme is at the repository root
    secrets:
      SHOPIFY_THEME_ACCESS_PASSWORD: ${{ secrets.SHOPIFY_THEME_ACCESS_PASSWORD }}
```

Commit it to the default branch (scheduled workflows run from the default branch only).

### 5. Test it

Open **Actions → Sync admin edits → Run workflow**. The run summary lists the changed files; if
there are any, a pull request titled **Sync theme updates** appears.

## Run CI on the sync pull request (recommended)

GitHub does not trigger other workflows for pull requests created with the default
`GITHUB_TOKEN`, so your theme check or tests will **not** run on the sync PR. To fix that, give the
workflow a different token. Pick one option:

**Option A: GitHub App (recommended; not tied to a person)**

1. Create a GitHub App with repository permissions **Contents: Read and write** and
   **Pull requests: Read and write**, then install it on your repositories.
2. Generate a private key.
3. Add the secret `SYNC_APP_PRIVATE_KEY` (the `.pem` content) and pass it with the App ID:

```yaml
    with:
      store: your-store
      app-id: "123456"
    secrets:
      SHOPIFY_THEME_ACCESS_PASSWORD: ${{ secrets.SHOPIFY_THEME_ACCESS_PASSWORD }}
      APP_PRIVATE_KEY: ${{ secrets.SYNC_APP_PRIVATE_KEY }}
```

**Option B: fine-grained personal access token**

Create a token with **Contents** and **Pull requests** read/write on the repository, store it as
`SYNC_PR_TOKEN`, and pass `PR_TOKEN: ${{ secrets.SYNC_PR_TOKEN }}` under `secrets`.

## Slack notifications

Post a message to Slack when the sync PR is opened or gets new changes. Runs where nothing
changed stay silent.

1. Create a Slack app at <https://api.slack.com/apps>, enable **Incoming Webhooks** and add a
   webhook for your channel (e.g. `#theme-edits`).
2. Store the webhook URL as the secret `SLACK_WEBHOOK_URL` (repository or organization).
3. Pass it to the workflow:

```yaml
    with:
      store: your-store
      slack-mention: "<!subteam^S0123>"   # optional: user group (<!subteam^ID>) or user (<@U0123>)
    secrets:
      SHOPIFY_THEME_ACCESS_PASSWORD: ${{ secrets.SHOPIFY_THEME_ACCESS_PASSWORD }}
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

The message looks like:

```
🛍️ Theme edits on your-store (live) · PR opened
Sync theme updates · your-org/your-repo
• themes/main/templates/index.json
• themes/main/sections/header-group.json
```

Up to 10 files are listed. A failed Slack request fails the run (the PR is already created by then).

For other channels (Discord, Teams, email), leave `SLACK_WEBHOOK_URL` unset and add your own job
that reads the workflow outputs:

```yaml
  notify:
    needs: sync
    if: contains(fromJSON('["created","updated"]'), needs.sync.outputs.pull-request-operation)
    runs-on: ubuntu-latest
    steps:
      - run: echo "PR ${{ needs.sync.outputs.pull-request-url }}"   # replace with your notifier
```

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `store` | (required) | Store handle or `*.myshopify.com` domain. |
| `theme-path` | `.` | Theme folder inside the repository. |
| `theme` | empty (live theme) | Theme ID or name to pull instead of the published theme. |
| `only` | see below | Newline-separated glob patterns to pull, relative to `theme-path`. Empty pulls every file. |
| `delete` | `false` | Delete local files (matching `only`) that no longer exist on the theme. |
| `branch` | `sync/admin-edits` | Branch the PR is opened from. |
| `base` | repository default branch | Base branch of the PR. |
| `pr-title` | `Sync theme updates` | Pull request title. |
| `commit-message` | `Sync theme updates from Shopify admin` | Commit message. |
| `labels` | empty | Labels to add to the PR (comma- or newline-separated). |
| `reviewers` | empty | GitHub usernames to request a review from. |
| `app-id` | empty | GitHub App ID, used with the `APP_PRIVATE_KEY` secret. |
| `cli-version` | `latest` | `@shopify/cli` version. |
| `slack-mention` | empty | Text appended to the Slack message, e.g. `<!subteam^S0123>`. |
| `slack-on` | `created,updated` | PR operations that post to Slack (`created`, `updated`, `closed`). |

Default `only`:

```
templates/*.json
templates/customers/*.json
sections/*.json
config/settings_data.json
locales/*.json
```

| Secret | Required | Description |
| --- | --- | --- |
| `SHOPIFY_THEME_ACCESS_PASSWORD` | yes | Theme Access password for the store. |
| `APP_PRIVATE_KEY` | no | Private key of the GitHub App given in `app-id`. |
| `PR_TOKEN` | no | Token used instead of a GitHub App. |
| `SLACK_WEBHOOK_URL` | no | Slack Incoming Webhook URL. No Slack message is sent without it. |

| Output | Description |
| --- | --- |
| `pull-request-url` | URL of the sync PR. |
| `pull-request-operation` | `created`, `updated`, `closed` or `none`. |
| `changed-files` | Newline-separated paths that differ from the base branch. |

## Things to know

- **Why not pull everything?** Between merging a feature and publishing its theme, the live theme
  is older than your default branch. Pulling every file would then show your new code as removed.
  Set `only: ""` if your team also edits Liquid/CSS in the admin code editor and you accept that noise.
- **Deleted files are ignored by default.** A template that exists in the repository but not yet on
  the live theme would otherwise be deleted in the PR. Set `delete: true` to mirror deletions.
- **Don't push to the sync branch.** It is rebuilt from the base branch on every run, so extra commits
  are lost. Merge the PR, then make follow-up changes on your own branch.
- **Schedule and cost.** Cron runs in UTC; convert from your local time zone. The example syncs hourly
  during Vietnam business hours on weekdays and every 4 hours on weekends: 73 runs a week of roughly
  one billed minute each, about 320 minutes per repository per month. Syncing every 30 minutes around
  the clock would cost about 1,500. Public repositories are free. GitHub may delay or drop scheduled
  runs under load, especially at the top of the hour, so the example runs at minute 5.
- **Several stores or themes.** Call the workflow once per store/theme, each with its own `branch`
  so their pull requests don't overwrite each other.
- **Versions.** Pin `@v1` for compatible updates, or a commit SHA for full control.

## License

MIT
