# Usage

This repository publishes reusable GitHub Actions workflows for monitoring app repositories.

Consumer repositories should reference workflow files in this repository by tag. For example:

```yaml
uses: qlik-oss/qlik-cloud-monitoring-app-workflows/.github/workflows/comment_checklist_pr.yml@workflows-v1.0.0
```

## Common requirements

- The caller repository should contain `release.json` at the repository root.
- The caller repository should store `.qvf` files in `assets/`.
- The `unbuild_app` workflow writes generated output to `diff/` and pushes changes back to the checked out branch.
- Consumer repositories should enable Dependabot updates for `github-actions` so workflow tag bumps are proposed automatically.

## Workflow: comment_checklist_pr

Posts a pull request checklist comment using the app name from `release.json`.

### Caller example

```yaml
name: Comment on PR

on:
  pull_request:
    types: [opened, synchronize]
    branches: ["main"]

jobs:
  verify:
    uses: qlik-oss/qlik-cloud-monitoring-app-workflows/.github/workflows/comment_checklist_pr.yml@workflows-v1.0.0
```

### Notes

- This workflow is intended for pull request events.
- The caller job must run in a repository that has a `release.json` file with a `name` property.
- The reusable workflow skips runs triggered by `dependabot[bot]` and `renovate[bot]`.

## Workflow: draft_release_on_main

Creates a draft GitHub release using the app name from `release.json`, all files in `assets/`, and any URLs listed in `release.json.externalAssets`.

### Caller example

```yaml
name: Build a draft release

on:
  workflow_dispatch:
    inputs:
      tag:
        description: Release tag (for example `v1.0.0`)
        required: true
        type: string

jobs:
  create-release:
    permissions:
      contents: write
    uses: qlik-oss/qlik-cloud-monitoring-app-workflows/.github/workflows/draft_release_on_main.yml@workflows-v1.0.0
    with:
      tag: ${{ inputs.tag }}
```

### Notes

- The caller repository should ensure `assets/` exists.
- `release.json.externalAssets` is optional. If present, each URL is downloaded into `assets/` before the release is drafted.
- The caller job needs `contents: write` permission.

## Workflow: unbuild_app

Imports each `.qvf` file from `assets/`, unbuilds it into `diff/`, downloads image assets, and pushes any generated diff changes back to the source branch.

### Caller example

```yaml
name: Unbuild app

on:
  pull_request:
    types: [opened, synchronize]
    branches: ["main"]
    paths:
      - "assets/**"

jobs:
  unbuild:
    permissions:
      contents: write
    uses: qlik-oss/qlik-cloud-monitoring-app-workflows/.github/workflows/unbuild_app.yml@workflows-v1.0.0
    secrets:
      QLIK_CLOUD_MONITORING_TENANT: ${{ secrets.QLIK_CLOUD_MONITORING_TENANT }}
      QLIK_CLOUD_MONITORING_OAUTH_ID: ${{ secrets.QLIK_CLOUD_MONITORING_OAUTH_ID }}
      QLIK_CLOUD_MONITORING_OAUTH_SECRET: ${{ secrets.QLIK_CLOUD_MONITORING_OAUTH_SECRET }}
```

### Optional manual caller example

Use this only if you want a manual run and you know which branch should receive the generated `diff/` commit.

```yaml
name: Unbuild app manually

on:
  workflow_dispatch:
    inputs:
      checkout_ref:
        description: Branch to update with generated diff content
        required: true
        type: string

jobs:
  unbuild:
    permissions:
      contents: write
    uses: qlik-oss/qlik-cloud-monitoring-app-workflows/.github/workflows/unbuild_app.yml@workflows-v1.0.0
    with:
      checkout_ref: ${{ inputs.checkout_ref }}
    secrets:
      QLIK_CLOUD_MONITORING_TENANT: ${{ secrets.QLIK_CLOUD_MONITORING_TENANT }}
      QLIK_CLOUD_MONITORING_OAUTH_ID: ${{ secrets.QLIK_CLOUD_MONITORING_OAUTH_ID }}
      QLIK_CLOUD_MONITORING_OAUTH_SECRET: ${{ secrets.QLIK_CLOUD_MONITORING_OAUTH_SECRET }}
```

### Notes

- Required secrets:
  - `QLIK_CLOUD_MONITORING_TENANT`
  - `QLIK_CLOUD_MONITORING_OAUTH_ID`
  - `QLIK_CLOUD_MONITORING_OAUTH_SECRET`
- The caller job needs `contents: write` permission because the workflow commits and pushes the generated `diff/` output.

## Versioning

- Publish releases from this repository using tags.
- Pin consumer repositories to a released tag such as `workflows-v1.0.0`.
- Update consumers by releasing a new tag here and allowing Dependabot to open version bump pull requests in downstream repositories.