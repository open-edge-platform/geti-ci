# Cleanup Old Container Images (composite)

This composite action deletes old container image versions from the GitHub Container Registry (GHCR) for a configurable set of packages, while always retaining release, release-candidate (RC) and `latest` images.

It replaces per-repository cleanup workflows (such as `cleanup-old-app-images.yml`) with a single shared implementation that is reused across repositories (e.g. [`geti`](https://github.com/open-edge-platform/geti), [`physical-ai-studio`](https://github.com/open-edge-platform/physical-ai-studio) and [`anomalib`](https://github.com/open-edge-platform/anomalib)).

## Features

- Configurable list of packages to clean up
- Retains release, RC and `latest` tagged images
- Keeps the most recent N non-release versions per package
- Dry-run mode (default) previews deletions without removing anything
- Writes a summary of the cleanup to the workflow run summary

## Retention rules

- Release images (tags matching semver `X.Y.Z`, e.g. `3.0.0`) are **never** deleted.
- Release-candidate images (tags matching `X.Y.ZrcN`, e.g. `3.0.0rc0`) are **never** deleted.
- The `latest` tag is **never** deleted.
- Among the remaining versions (dev/daily builds and untagged versions) the most recent `min-versions-to-keep` are kept; older ones are deleted.

## Usage

> [!IMPORTANT]
> This is a monorepo containing several Actions. When we release the Cleanup Old Container Images action, we create a tag `cleanup-images/v<version>`, e.g. `cleanup-images/v0.1.0`.

> [!NOTE]
> The default `GITHUB_TOKEN` may not have sufficient scopes to delete package versions in some contexts (e.g. forked PRs). Provide a token with `packages:write` permission via the `token` input if needed.

Example scheduled cleanup with a dry-run default and manual override:

```yaml
name: Cleanup Old Application Container Images

on:
  workflow_dispatch:
    inputs:
      min_versions_to_keep:
        description: "Number of most recent non-release image versions to keep per package"
        required: false
        type: number
        default: 10
      dry_run:
        description: "Preview deletions without removing image versions"
        required: false
        type: boolean
        default: true
  schedule:
    # Run weekly on Sunday at 03:00 UTC.
    - cron: "0 3 * * 0"

permissions: {}

jobs:
  cleanup-old-images:
    name: Delete old GHCR application images
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - name: Cleanup old container images
        uses: open-edge-platform/geti-ci/actions/cleanup-images@<SHA> # cleanup-images/v0.1.0
        with:
          packages: |
            geti-cpu
            geti-xpu
            geti-cuda
          min-versions-to-keep: ${{ inputs.min_versions_to_keep || 10 }}
          dry-run: ${{ inputs.dry_run == false && 'false' || 'true' }}
```

## Inputs

| Name                   | Type    | Description                                                                                           | Default                          | Required |
| ---------------------- | ------- | ----------------------------------------------------------------------------------------------------- | -------------------------------- | -------- |
| `packages`             | String  | Space- or newline-separated list of GHCR container package names to clean up                          | —                                | Yes      |
| `min-versions-to-keep` | String  | Number of most recent non-release image versions to keep per package                                  | `10`                             | No       |
| `dry-run`              | String  | Preview deletions without removing image versions. Must be `true` or `false`                          | `true`                           | No       |
| `owner`                | String  | GHCR package owner (user or organization)                                                             | `${{ github.repository_owner }}` | No       |
| `token`                | String  | Token with `packages:write` permission used to list and delete package versions                       | `${{ github.token }}`            | No       |

## Outputs

| Name               | Type   | Description                                        |
| ------------------ | ------ | -------------------------------------------------- |
| `scanned-packages` | String | Number of packages scanned                         |
| `retained-release` | String | Number of release versions retained                |
| `candidates`       | String | Number of versions matched for deletion            |
| `deleted`          | String | Number of versions deleted (`0` in dry-run mode)   |

## Required permissions

The job using this action must grant `packages: write` so that package versions can be listed and deleted:

```yaml
permissions:
  packages: write
```
