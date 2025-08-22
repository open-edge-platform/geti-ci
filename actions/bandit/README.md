# Bandit (composite)

This composite action executes GitHub Actions workflows scanning using [Bandit](https://github.com/PyCQA/bandit), providing configurable security analysis capabilities.

## Usage

Example usage in a repository on PR (checks only changed files):

```yaml
name: Bandit scan

on:
  pull_request:

jobs:
  Bandit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Bandit scan
        uses: ./actions/bandit
        with:
          scan-scope: changed
          severity-level: MEDIUM
          confidence-level: HIGH
          fail-on-findings: true
```

Example usage in a repository on schedule (checks all scope), uploads results in SARIF format:

```yaml
name: Bandit scan

on:
  schedule:
    - cron: "0 2 * * *"

permissions:
  contents: read
  security-events: write # to upload sarif output

jobs:
  Bandit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Bandit scan
        uses: ./actions/bandit
        with:
          scan-scope: all
          severity-level: LOW
          confidence-level: LOW
          fail-on-findings: false
```

## Inputs

| Name               | Type    | Description                                          | Default Value       | Required |
| ------------------ | ------- | ---------------------------------------------------- | ------------------- | -------- |
| `scan-scope`       | String  | Scope of files to scan (all/changed)                 | `changed`           | No       |
| `paths`            | String  | Paths to scan when using all scope                   | `.`                 | No       |
| `severity-level`   | String  | Minimum severity level to report (LOW/MEDIUM/HIGH)   | `LOW`               | No       |
| `confidence-level` | String  | Minimum confidence level to report (LOW/MEDIUM/HIGH) | `LOW`               | No       |
| `output-format`    | String  | Format for scan results (plain/json/sarif)           | `sarif`             | No       |
| `fail-on-findings` | boolean | Whether to fail the action if issues are found       | `true`              | No       |
| `bandit-version`   | String  | Bandit version                                       | Updated by Renovate | No       |

## Outputs

| Name          | Type   | Description                       |
| ------------- | ------ | --------------------------------- |
| `scan_result` | String | Exit code of the Bandit scan      |
| `report_path` | String | Path to the generated report file |

## Required permissions

This composite action requires `security-events: write` to upload SARIF results into Security tab.
