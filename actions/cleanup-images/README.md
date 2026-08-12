# Cleanup Old Container Images (composite)

This composite action deletes old container image versions from the GitHub Container Registry (GHCR) for a configurable set of packages, while always retaining release, release-candidate (RC) and `latest` images.

It replaces per-repository cleanup workflows (such as `cleanup-old-app-images.yml`) with a single shared implementation that is reused across repositories (e.g. [`geti`](https://github.com/open-edge-platform/geti), [`physical-ai-studio`](https://github.com/open-edge-platform/physical-ai-studio) and [`anomalib`](https://github.com/open-edge-platform/anomalib)).

## Features

- Configurable list of packages to clean up
- Retains release, RC and `latest` tagged images
- Keeps the most recent N non-release versions per package
- Dry-run mode (default) previews deletions without removing anything
- Writes a summary of the cleanup to the workflow run summary

## Prerequisites

The action runs a Bash script and shells out to a few tools that must be available on the runner (all preinstalled on GitHub-hosted `ubuntu-*` runners; install them yourself on custom/self-hosted runners):

- **`bash`** — the action step uses `shell: bash`.
- **[`gh`](https://cli.github.com/) (GitHub CLI) 2.x+** — lists and deletes package versions via the GitHub API. It reads the token from the `GH_TOKEN` environment variable, which the action sets from the `token` input.
- **[`jq`](https://jqlang.github.io/jq/) 1.6+** — parses API responses and computes the retention filter (uses `strptime`/`mktime`, `unique`, `@uri`).
- **[`curl`](https://curl.se/) 7.x+** — resolves retained image manifests and OCI referrers from the GHCR registry to protect child manifests and cosign signatures.

The registry uses the OCI [Referrers API](https://github.com/opencontainers/distribution-spec/blob/main/spec.md#listing-referrers) to discover cosign artifacts; GHCR supports it. If a runner cannot reach `ghcr.io` or the tools are missing, manifest resolution is skipped with a warning and only tag-based protection applies.

## Retention rules

- Release images (tags matching semver `X.Y.Z`, e.g. `3.0.0`) are **never** deleted.
- Release-candidate images (tags matching `X.Y.ZrcN`, e.g. `3.0.0rc0`) are **never** deleted.
- The `latest` tag is **never** deleted.
- Any tag listed in `extra-retained-tags` (e.g. `develop`) is **never** deleted.
- Untagged child manifests referenced by a retained index (per-arch layers, buildx provenance/SBOM attestations) and cosign signatures/attestations of retained images are **never** deleted, so retained tags stay pullable.
- Among the remaining versions (dev/daily builds and unreferenced untagged versions) the most recent `min-versions-to-keep` are kept; older ones are deleted.

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
| `extra-retained-tags`  | String  | Space- or newline-separated additional tags to never delete (e.g. `develop`). Their child manifests are protected too | `""`                             | No       |
| `owner`                | String  | GHCR package owner (user or organization)                                                             | `${{ github.repository_owner }}` | No       |
| `token`                | String  | Token with `packages:write` permission used to list and delete package versions                       | `${{ github.token }}`            | No       |

## Outputs

| Name               | Type   | Description                                        |
| ------------------ | ------ | -------------------------------------------------- |
| `scanned-packages` | String | Number of packages scanned                         |
| `retained-release` | String | Number of release versions retained                |
| `candidates`       | String | Number of versions matched for deletion            |
| `deleted`          | String | Number of versions deleted (`0` in dry-run mode)   |
| `candidate-digests`| String | Space-separated digests of versions matched for deletion |

## Required permissions

The job using this action must grant `packages: write` so that package versions can be listed and deleted:

```yaml
permissions:
  packages: write
```
