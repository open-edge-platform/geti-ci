# Trivy Scan Diff (composite)

Scans a target container image and a digest-pinned base image with [Trivy](https://github.com/aquasecurity/trivy), then diffs the vulnerability findings into two report sections:

- Non-base image CVEs - introduced by the application layers on top of the base image
- Base image CVEs - inherited from the base image itself

It also reports the base image's publication timestamp (best-effort, read from its image config) so you can judge for yourself whether the pinned base image is current.

Both sections are written to `diff-report.md`/`diff-report.json` (uploaded as an artifact), but the job summary (`$GITHUB_STEP_SUMMARY`) only includes the non-base image CVEs section and the base image's publication date, since base-image CVEs aren't actionable here.

This is a scan-and-report action: it never fails the workflow based on CVE findings or CVE severity.

## Usage

```yaml
name: Trivy scan diff

...

jobs:
  trivy-scan-diff:
    runs-on: ubuntu-latest
    steps:
      - name: Run Trivy scan diff
        id: trivy-scan-diff
        uses: open-edge-platform/geti-ci/actions/trivy-scan-diff@<SHA> # trivy-scan-diff/v0.1.0
        with:
          image: "myregistry.io/myapp:latest"
          base-image: "python@sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcd"
          severity: "MEDIUM,HIGH,CRITICAL"
```

## Usage in a matrix (scanning several images)

`actions/upload-artifact` requires a unique artifact name per workflow run, so each matrix leg **must** set a unique `artifact-name`. When scanning several images against the same base image, sanitize the image reference into a safe artifact name:

```yaml
jobs:
  trivy-scan-diff:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        image:
          - registry/app-a:latest
          - registry/app-b:latest
    steps:
      - name: Sanitize image name for artifact
        id: sanitize
        shell: bash
        env:
          IMAGE: ${{ matrix.image }}
        run: |
          SAFE_NAME=$(echo "$IMAGE" | tr -c 'a-zA-Z0-9._-' '-')
          echo "artifact_name=trivy-scan-diff-${SAFE_NAME}" >> "$GITHUB_OUTPUT"

      - uses: open-edge-platform/geti-ci/actions/trivy-scan-diff@<SHA> # trivy-scan-diff/v0.1.0
        with:
          image: ${{ matrix.image }}
          base-image: python@sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcd
          artifact-name: ${{ steps.sanitize.outputs.artifact_name }}
```

If each image needs a different base image, use `matrix.include` instead and reference `matrix.base-image`:

```yaml
strategy:
  fail-fast: false
  matrix:
    include:
      - image: registry/app-a:latest
        base-image: python@sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcd
      - image: registry/app-b:latest
        base-image: node@sha256:abcdef0123456789abcdef0123456789abcdef0123456789abcdef01234567
```

Job outputs (`non_base_cve_count`, `base_cve_count`, etc.) are per matrix leg, so GitHub Actions does not auto-aggregate matrix job outputs. If a combined total across all images is needed, aggregate via `needs.<job>.outputs.*` in a follow-up summary job.

## Inputs

| Name                | Type    | Description                                                                                     | Default Value                        | Required |
| -------------------- | ------- | ------------------------------------------------------------------------------------------------| --------------------------------------| -------- |
| `image`              | String  | Target container image to scan                                                                  | —                                     | Yes      |
| `base-image`         | String  | Base image reference, must be pinned by digest (`registry/repo@sha256:...`)                      | —                                     | Yes      |
| `severity`           | String  | Severity levels to check, comma-separated (UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL)                     | `UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL` | No       |
| `ignore_unfixed`     | boolean | Ignore unpatched/unfixed vulnerabilities                                                         | `false`                               | No       |
| `timeout`            | String  | Trivy timeout duration per scan (e.g. 5m, 10m)                                                   | `10m`                                 | No       |
| `trivy-version`      | String  | Trivy version                                                                                     | Updated by Renovate                   | No       |
| `artifact-name`      | String  | Upload artifact name. Must be unique per matrix leg when used inside a `strategy.matrix`          | `trivy-scan-diff-results`             | No       |

`base-image` can reference any registry (Docker Hub, `ghcr.io`, `quay.io`, etc.), as long as it includes a `@sha256:...` digest.

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
| `base_image_created`             | String | Best-effort publication timestamp (RFC3339) of `base-image`, read from its image config. Empty if it could not be determined. Shown as a `YYYY-MM-DD` date in the markdown report and job summary. |

The uploaded artifact (under `artifact-name`) contains `image-report.json`, `image-report.txt`, `base-image-report.json`, `base-image-report.txt`, `diff-report.md`, and `diff-report.json`.
