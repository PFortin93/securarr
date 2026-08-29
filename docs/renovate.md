# Renovate

Every image in this repo carries an explicit version tag on the
assumption that *something* is watching for updates. That something is
[Renovate](https://docs.renovatebot.com/), configured by
`renovate.json5`: grouped PRs for the noisy linuxserver images,
individual PRs for the load-bearing ones (gluetun, Plex, Caddy), digest
pinning on top of the tags, and **no automerge anywhere** — every merge
is a human action, by supply-chain policy. That matters double if you
adopt the Flatcar sync loop, where merging to `main` *is* deploying.

There are two ways to run it. Either way, expect a burst of PRs on the
first run — that is `pinDigests` adding digests to every image tag, not
noise. After the first wave it settles into a trickle.

## Default: the hosted app

Install the [Renovate GitHub App](https://github.com/apps/renovate) on
your fork and you are done. Mend runs Renovate on their infrastructure
on their cadence; `renovate.json5` is still the config it reads, so the
whole update policy carries over. No token, no secret, no workflow —
and nothing that breaks on a fresh fork.

Run logs live in the [Mend developer portal](https://developer.mend.io/)
rather than your Actions tab. If you want a slower cadence than the
app's default, add a `schedule` block to `renovate.json5`.

## Alternative: entirely on GitHub, via your own GitHub App

If you would rather keep the runs inside your own repository's Actions
— logs next to your other CI, cadence owned by your cron — self-host
the runner with a personal GitHub App for authentication. This is what
Renovate's own docs recommend over a PAT: installation tokens live one
hour instead of sitting in a secret for a year, nothing ever needs
rotating, and PRs come from `your-app[bot]` instead of your user
account.

### 1. Create the app

GitHub → Settings → Developer settings → GitHub Apps → New GitHub App.

- **Name**: anything unique; it becomes the bot's name on your PRs.
- **Webhook**: untick Active — no webhook is needed.
- **Repository permissions**:

  | Permission | Access | Why |
  |---|---|---|
  | Contents | Read and write | push the update branches |
  | Pull requests | Read and write | open the PRs |
  | Workflows | Read and write | Renovate also updates the pinned actions and tool images under `.github/workflows/`, and GitHub rejects pushes touching workflow files from credentials without this |

- **Where can this app be installed**: Only on this account.

After creating: note the **App ID**, then generate a **private key**
(downloads a `.pem`) — that file is the credential; treat it like one.

### 2. Install it and store the credentials

Install the app on the repo (the app's page → Install App), then in the
repo settings add:

- a repository **variable** `RENOVATE_APP_ID` — the App ID (not secret)
- a repository **secret** `RENOVATE_APP_PRIVATE_KEY` — the full `.pem`
  contents

### 3. Add the workflow

`.github/workflows/renovate.yml`:

```yaml
# Self-hosted Renovate, authenticating as a personal GitHub App. The
# create-github-app-token step mints a one-hour installation token per
# run; no long-lived credential exists beyond the app's private key.
name: renovate
on:
  schedule:
    - cron: '0 12 * * *'   # daily
  workflow_dispatch:
permissions:
  contents: read
jobs:
  renovate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false
      - uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
        id: app-token
        with:
          app-id: ${{ vars.RENOVATE_APP_ID }}
          private-key: ${{ secrets.RENOVATE_APP_PRIVATE_KEY }}
      - uses: renovatebot/github-action@a889a8abcb11ef7feaafaf5e483ea01d4bf7774e # v43.0.5
        with:
          token: ${{ steps.app-token.outputs.token }}
        env:
          RENOVATE_REPOSITORIES: ${{ github.repository }}
          # Commits and PRs attributed to the app's bot identity. The
          # numeric id comes from:
          #   gh api '/users/<app-slug>%5Bbot%5D' --jq .id
          RENOVATE_USERNAME: '<app-slug>[bot]'
          RENOVATE_GIT_AUTHOR: '<app-slug>[bot] <<bot-user-id>+<app-slug>[bot]@users.noreply.github.com>'
          LOG_LEVEL: info
```

Replace `<app-slug>` (the lowercased app name from its URL) and
`<bot-user-id>`, then trigger a first run with:

```bash
gh workflow run renovate
```

Renovate will keep this workflow's own pins current too — that is what
the Workflows permission and the `custom.regex` manager in
`renovate.json5` are for.

### Why not the built-in GITHUB_TOKEN

Two reasons it cannot work: it cannot push changes to workflow files at
all, and PRs it creates **do not trigger other workflows** — so the
validate and scan gates would silently never run against Renovate's
PRs. An update policy whose PRs skip CI is worse than no automation, so
Renovate's docs rule it out entirely.

### Why not a PAT

It works (Contents + Pull requests + Workflows, read/write), but it is
strictly worse than the app: fine-grained PATs expire within a year and
fail silently when they do, the token sits in a secret for its whole
lifetime, and every PR is attributed to your personal account.
