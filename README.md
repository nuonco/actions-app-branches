# actions-app-branches

GitHub Actions for triggering a [Nuon app branch](https://docs.nuon.co) run or preview.

Each action installs the Nuon CLI, writes config, and runs the same preflight as [actions-nuon](https://github.com/nuonco/actions-nuon). The trigger and preview steps are stubbed: they print the Nuon command and do not execute it yet.

| Action | Command |
| ------ | ------- |
| `branches/trigger/tag` | `nuon branches trigger --run-type tag` |
| `branches/trigger/commit` | `nuon branches trigger --run-type commit` |
| `branches/trigger/pr` | `nuon branches trigger --run-type pr` |
| `branches/preview/pr` | `nuon branches preview --pr-number` |
| `branches/preview/git-ref` | `nuon branches preview --git-ref` |

## Authentication

Omit `api_token` to authenticate with [OIDC federation](https://docs.nuon.co/concepts/oidc-federation). The job needs `id-token: write`. The CLI requests an OIDC token whose audience defaults to `api_url`. Set `oidc_audience` when the trust policy uses a different audience.

Otherwise pass an [API token](https://docs.nuon.co/concepts/api-tokens) via `api_token`.

## Usage

### Trigger from a tag

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: nuonco/actions-app-branches/branches/trigger/tag@v1
        with:
          org_id: ${{ vars.NUON_ORG_ID }}
          app_id: ${{ vars.NUON_APP_ID }}
          branch_id: production
          tag: ${{ github.ref_name }}
```

### Trigger from a commit

```yaml
- uses: nuonco/actions-app-branches/branches/trigger/commit@v1
  with:
    org_id: ${{ vars.NUON_ORG_ID }}
    app_id: ${{ vars.NUON_APP_ID }}
    branch_id: production
    sha: ${{ github.sha }}
```

### Trigger from a pull request

```yaml
- uses: nuonco/actions-app-branches/branches/trigger/pr@v1
  with:
    org_id: ${{ vars.NUON_ORG_ID }}
    app_id: ${{ vars.NUON_APP_ID }}
    branch_id: production
    pr_number: ${{ github.event.pull_request.number }}
```

### Preview a pull request

```yaml
- uses: nuonco/actions-app-branches/branches/preview/pr@v1
  with:
    org_id: ${{ vars.NUON_ORG_ID }}
    app_id: ${{ vars.NUON_APP_ID }}
    branch_id: preview
    pr_number: ${{ github.event.pull_request.number }}
    mode: plan-only
    install_id: ${{ vars.NUON_INSTALL_ID }}
```

### Preview a git ref

```yaml
- uses: nuonco/actions-app-branches/branches/preview/git-ref@v1
  with:
    org_id: ${{ vars.NUON_ORG_ID }}
    app_id: ${{ vars.NUON_APP_ID }}
    branch_id: preview
    git_ref: ${{ github.head_ref }}
    mode: apply
    install_id: ${{ vars.NUON_INSTALL_ID }}
```

## Inputs

Shared by every action:

| Input | Description | Required | Default |
| ----- | ----------- | -------- | ------- |
| `org_id` | Nuon organization ID | Yes | - |
| `api_token` | Nuon API token. Omit to use OIDC federation. | No | - |
| `app_id` | Nuon app ID written into the CLI config | No | - |
| `api_url` | Nuon API URL | No | `https://api.nuon.co` |
| `oidc_audience` | OIDC audience when it differs from `api_url` | No | `api_url` |
| `nuon_version` | CLI version. `latest` resolves from `api_url` | No | `latest` |
| `branch_id` | App branch ID or name | Yes | - |
| `force` | Pass `--force` to rebuild all components | No | `false` |
| `no_wait` | Pass `--no-wait` and skip the workflow viewer | No | `true` |

Trigger ref inputs:

| Action | Input | Description |
| ------ | ----- | ----------- |
| `branches/trigger/tag` | `tag` | Git tag name |
| `branches/trigger/commit` | `sha` | Commit SHA |
| `branches/trigger/pr` | `pr_number` | Pull request number (positive integer) |

Preview inputs:

| Input | Description | Required | Default |
| ----- | ----------- | -------- | ------- |
| `pr_number` | Pull request number. `branches/preview/pr` only. | Yes | - |
| `git_ref` | Git branch to preview. `branches/preview/git-ref` only. | Yes | - |
| `mode` | `plan-only`, `apply`, or `build-only`. Omit to use the branch config, then `plan-only`. | No | - |
| `install_id` | Install to run the preview against | No | - |
| `head_sha` | Commit SHA for the preview source | No | - |
| `config_id` | Branch config ID. Omit to use the latest. | No | - |
| `auto_approve` | Skip the approval gate before deploy steps | No | `false` |
| `wait` | Block until the preview workflow completes. Takes precedence over `no_wait`. | No | `false` |

## Outputs

| Output | Description |
| ------ | ----------- |
| `nuon_version` | CLI version selected for this run |

## How it works

1. **Version** resolves the CLI version, including the release-bucket fallback from actions-nuon.
2. **Install** downloads the Nuon CLI.
3. **Config** writes `~/.nuon` and exports `NUON_CONFIG_FILE`.
4. **Preflight** runs `nuon orgs get`, and `nuon apps get` when `app_id` is set.
5. **Trigger or preview** calls `scripts/trigger.sh` or `scripts/preview.sh`. Those functions build the Nuon command and currently only print it.

`NUON_*` environment variables on a later step override the generated config. The CLI stays on `PATH` for steps after the action.
