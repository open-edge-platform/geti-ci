# Bandit (composite)

This composite action executes Python security scanning using [Bandit](https://github.com/PyCQA/bandit), providing configurable security analysis with SARIF reporting capabilities.

## Features

- Automatically handles SARIF upload limitations for private repositories
- Support for both changed files (PR context) and full repository scans
- Customize severity and confidence levels
- Adds GitHub-compatible security-severity properties for better integration

## Minimal severity and confidence levels

For unification purposes, we use the following values as minimal severity and minimal confidence levels across all applicable actions:

- **severity-level**: `LOW`,`MEDIUM`,`HIGH`,`CRITICAL`
- **confidence-level**: `LOW`,`MEDIUM`,`HIGH`

Each action converts these input unified severity and confidence levels into the severity and confidence levels supported by the specific tool used by that action.
This is necessary for unification across security tools that produce and upload SARIF output.

GitHub [supports](https://github.blog/changelog/2021-07-19-codeql-code-scanning-new-severity-levels-for-security-alerts/) an additional security-severity property in SARIF (this is used by CodeQL and Trivy).
Each action enriches the original SARIF report produced by the tool with this security-severity property based on SARIF alert levels. Numeric values are used to make the following additions based on level:

- `none` → `none` (0.0)
- `note` → `low` (3.0)
- `warning` → `medium` (5.0)
- `error` → `high` (8.0)

**Mapping flow**: Action input unified severity level → severity level used by the tool → SARIF level used by the tool → security-severity added by action into original SARIF report

### Bandit-specific mappings

Bandit supports:

- severity levels: `low`, `medium`, `high`
- confidence levels: `low`, `medium`, `high`

The following tables summarize minimal severity levels mapping approach for Bandit.

| Action input severity → | Bandit severity → | Bandit SARIF level → | Action security-severity |
| ----------------------- | ----------------- | -------------------- | ------------------------ |
| `LOW`                   | `low`             | `note`               | `low`                    |
| `MEDIUM`                | `medium`          | `warning`            | `medium`                 |
| `HIGH`                  | `high`            | `error`              | `high`                   |
| `CRITICAL`              | `high`            | `error`              | `high`                   |

For Bandit severity to SARIF level mapping see Bandit [code](https://github.com/PyCQA/bandit/blob/main/bandit/formatters/sarif.py).

The following tables summarize minimal confidence level mapping approach for Bandit (1:1 mapping).

| Action input confidence → | Bandit confidence |
| ------------------------- | ----------------- |
| `LOW`                     | `low`             |
| `MEDIUM`                  | `medium`          |
| `HIGH`                    | `high`            |

## Workflow Context Requirements

For proper SARIF upload and PR commenting, this action **MUST** be used within the same workflow and job where GitHub Advanced Security expects results. GitHub Code Scanning correlates SARIF uploads with workflow/job context to provide PR comments and security alerts. Using different workflows or jobs will break this correlation.

## Usage

Example usage that scans changed files on PRs and all files on push/schedule:

```yaml
name: Bandit scan

on:
  pull_request:
  push:
    branches:
      - main
  schedule:
    - cron: "0 2 * * *"

permissions:
  contents: read
  security-events: write # Required for SARIF upload

jobs:
  Bandit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Run Bandit scan
        uses: ./actions/bandit
        with:
          scan-scope: ${{ github.event_name == 'pull_request' && 'changed' || 'all' }}
          severity-level: MEDIUM
          confidence-level: HIGH
          fail-on-findings: true
```

## Inputs

| Name               | Type    | Description                                                 | Default Value       | Required |
| ------------------ | ------- | ----------------------------------------------------------- | ------------------- | -------- |
| `artifact-name`    | String  | Artifact name                                               | `bandit-results`    | No       |
| `scan-scope`       | String  | Scope of files to scan (all/changed)                        | `changed`           | No       |
| `paths`            | String  | Paths to scan when using all scope                          | `.`                 | No       |
| `config_file`      | String  | Path to pyproject.toml or custom bandit config              | `pyproject.toml`    | No       |
| `severity-level`   | String  | Minimum severity level to report (LOW/MEDIUM/HIGH/CRITICAL) | `LOW`               | No       |
| `confidence-level` | String  | Minimum confidence level to report (LOW/MEDIUM/HIGH)        | `LOW`               | No       |
| `output-format`    | String  | Format for scan results (json/txt/html/csv/sarif)           | `txt`               | No       |
| `fail-on-findings` | Boolean | Whether to fail the action if issues are found              | `true`              | No       |
| `upload-sarif`     | Boolean | Whether to upload SARIF results to GitHub Security          | `true`              | No       |
| `bandit-version`   | String  | Bandit version                                              | Updated by Renovate | No       |

## Outputs

| Name          | Type   | Description                       |
| ------------- | ------ | --------------------------------- |
| `scan_result` | String | Exit code of the Bandit scan      |
| `report_path` | String | Path to the generated report file |

**Note**: For private repositories without Advanced Security subscription, SARIF upload is automatically skipped, but reports are still available as workflow artifacts or in workflow logs.

## Required permissions

This composite action requires `security-events: write` to upload SARIF results into the Security tab.
