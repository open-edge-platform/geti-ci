# Semgrep (composite)

This composite action executes GitHub Actions workflows scanning using [Semgrep](https://github.com/semgrep/semgrep), providing configurable security analysis capabilities.

## Usage

Example usage in a repository on PR (checks only changed files):

```yaml
name: Semgrep scan

on:
  pull_request:

jobs:
  semgrep-on-pr:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@08c6903cd8c0fde910a37f88322edcfb5dd907a8 # v5.0.0
        with:
          persist-credentials: false
          fetch-depth: 0 # <--- for diff-aware scan
      - name: Run Semgrep scan
        uses: open-edge-platform/geti-ci/actions/semgrep@552c4...
        with:
          scan-scope: changed
          output-format: sarif
          severity: CRITICAL
          fail-on-findings: true
```
On changed scope (i.e., check on PR), action outputs results in a text format into console and duplicates it into SARIF file with `--sarif-output`
SARIF results will be uploaded into PR comments for public repos, `--baseline-commit` is used to trigger diff-aware scan.

Example usage in a repository on schedule (checks all scope), uploads results in SARIF format:

```yaml
name: Semgrep scan

on:
  schedule:
    - cron: "0 2 * * *"

permissions:
  contents: read
  security-events: write # to upload sarif output

jobs:
  semgrep-on-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@08c6903cd8c0fde910a37f88322edcfb5dd907a8 # v5.0.0
        with:
          persist-credentials: false
      - name: Run Semgrep scan
        uses: open-edge-platform/geti-ci/actions/semgrep@552c4...
        with:
          scan-scope: all
          output-format: sarif
          severity: LOW
          fail-on-findings: false
```

## Inputs

| Name               | Type    | Description                                        | Default Value                                          | Required |
| ------------------ | ------- | -------------------------------------------------- | ------------------------------------------------------ | -------- |
| `scan-scope`       | String  | Scope of files to scan (all/changed)               | `changed`                                              | No       |
| `paths`            | String  | Paths to scan when using all scope                 | `.`                                                    | No       |
| `severity`         | String  | Minimum severity level to report (LOW/MEDIUM/HIGH) | `LOW`                                                  | No       |
| `config`           | String  | Semgrep rules or config to use                     | `p/default p/cwe-top-25 p/trailofbits p/owasp-top-ten` | No       |
| `output-format`    | String  | Format for scan results (text/json/sarif)          | `sarif`                                                | No       |
| `fail-on-findings` | boolean | Whether to fail the action if issues are found     | `true`                                                 | No       |
| `timeout`          | String  | Maximum time to run semgrep in seconds             | `300`                                                  | No       |
| `timeout`          | String  | Maximum time to run semgrep in seconds             | `300`                                                  | No       |
| `semgrep-version`  | String  | Semgrep version                                    | Updated by Renovate                                    | No       |

## Outputs

| Name          | Type   | Description                       |
| ------------- | ------ | --------------------------------- |
| `scan_result` | String | Exit code of the Semgrep scan     |
| `report_path` | String | Path to the generated report file |

## Required permissions

This composite action requires `security-events: write` to upload SARIF results into Security tab.
