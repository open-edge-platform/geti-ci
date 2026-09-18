# Trivy Scan Diff (composite)

Scans a target container image and a digest-pinned base image with [Trivy](https://github.com/aquasecurity/trivy), then diffs the vulnerability findings into two report sections:

- **Non-base image CVEs** — introduced by the application layers on top of the base image
- **Base image CVEs** — inherited from the base image itself

It also performs a best-effort check of whether the pinned `base-image` digest matches the registry's `latest` tag (or highest semver-looking tag, if no `latest` tag exists).

This is a **scan-and-report** action: it never fails the workflow based on CVE findings, CVE severity, or an outdated base image. The only failure case is an unpinned `base-image` input — it must always be pinned by digest (`registry/repo@sha256:...`) so the diff and latest-check are reproducible.

## Usage

```yaml
name: Trivy scan diff

on:
  pull_request:

permissions:
  contents: read

jobs:
  trivy-scan-diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - name: Run Trivy scan diff
        id: trivy-scan-diff
        uses: ./actions/trivy-scan-diff
        with:
          image: "myregistry.io/myapp:latest"
          base-image: "python@sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcd"
          severity: "MEDIUM,HIGH,CRITICAL"
```

## Usage in a matrix (scanning several images)

`actions/upload-artifact` requires a unique artifact name per workflow run, so each matrix leg **must** set a unique `artifact-name`:

```yaml
jobs:
  trivy-scan-diff:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        include:
          - image: registry/app-a:latest
            base-image: python@sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcd
          - image: registry/app-b:latest
            base-image: node@sha256:abcdef0123456789abcdef0123456789abcdef0123456789abcdef01234567
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - uses: ./actions/trivy-scan-diff
        with:
          image: ${{ matrix.image }}
          base-image: ${{ matrix.base-image }}
          artifact-name: trivy-scan-diff-${{ strategy.job-index }}
```

Job outputs (`non_base_cve_count`, `base_cve_count`, etc.) are per matrix leg — GitHub Actions does not auto-aggregate matrix job outputs. If a combined total across all images is needed, aggregate via `needs.<job>.outputs.*` in a follow-up summary job.

## Inputs

| Name                | Type    | Description                                                                                     | Default Value                        | Required |
| -------------------- | ------- | ------------------------------------------------------------------------------------------------| --------------------------------------| -------- |
| `image`              | String  | Target container image to scan                                                                  | —                                     | Yes      |
| `base-image`         | String  | Base image reference, must be pinned by digest (`registry/repo@sha256:...`)                      | —                                     | Yes      |
| `severity`           | String  | Severity levels to check, comma-separated (UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL)                     | `UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL` | No       |
| `ignore_unfixed`     | boolean | Ignore unpatched/unfixed vulnerabilities                                                         | `false`                               | No       |
| `check-base-latest`  | boolean | Best-effort check whether `base-image`'s digest matches the registry's latest tag/version        | `true`                                | No       |
| `timeout`            | String  | Trivy timeout duration per scan (e.g. 5m, 10m)                                                   | `10m`                                 | No       |
| `trivy-version`      | String  | Trivy version                                                                                     | Updated by Renovate                   | No       |
| `artifact-name`      | String  | Upload artifact name. Must be unique per matrix leg when used inside a `strategy.matrix`          | `trivy-scan-diff-results`             | No       |

`base-image` can reference any registry (Docker Hub, `ghcr.io`, `quay.io`, etc.), as long as it includes a `@sha256:...` digest. Scanning a private `image`/`base-image` requires the caller to authenticate beforehand (e.g. via `docker/login-action`) so Trivy and `crane` can read `~/.docker/config.json`.

## Outputs

| Name                           | Type   | Description                                                                     |
| ------------------------------- | ------ | -------------------------------------------------------------------------------|
| `scan_result`                   | String | Execution status of the scan+diff pipeline (`0` = success). Never reflects CVE findings. |
| `report_path`                    | String | Path to the generated markdown diff report (2 sections)                        |
| `json_report_path`               | String | Path to the generated JSON diff report                                         |
| `image_scan_report_path`         | String | Path to the full plain-text Trivy scan of `image`                              |
| `base_image_scan_report_path`    | String | Path to the full plain-text Trivy scan of `base-image`                         |
| `non_base_cve_count`             | String | Count of CVEs introduced by app layers (not present in base)                    |
| `base_cve_count`                  | String | Count of CVEs inherited from the base image                                    |
| `base_image_is_latest`           | String | `true` / `false` / `unknown` (best-effort)                                      |

The uploaded artifact (under `artifact-name`) contains `image-report.json`, `image-report.txt`, `base-image-report.json`, `base-image-report.txt`, `diff-report.md`, and `diff-report.json`.
