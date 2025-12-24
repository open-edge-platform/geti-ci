# Zizmor (composite)

This composite action executes GitHub Actions workflows scanning using [Zizmor](https://github.com/zizmorcore/zizmor), providing configurable security analysis with SARIF reporting capabilities.

## Features

- Scans GitHub Actions workflow files for security issues
- Supports two scan scopes: `all` (full scan) or `changed` (changed files only)
- Generates both SARIF reports (for GitHub Security tab) and plain/json reports
- Uploads SARIF to Code Scanning for public repositories
- Uploads all reports as workflow artifacts
- Configurable severity/confidence thresholds and failure behavior

## Important: Workflow Context Requirements

For proper SARIF upload and PR commenting, this action **MUST** be used within the same workflow and job where GitHub Advanced Security expects results. GitHub Code Scanning correlates SARIF uploads with workflow/job context to provide PR comments and security alerts. Using different workflows or jobs will break this correlation.

## Usage

Example usage that scans changed files on PRs and all files on push/schedule:

```yaml
name: Zizmor scan

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
  zizmor:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Run Zizmor scan
        uses: ./actions/zizmor
        with:
          scan-scope: ${{ github.event_name == 'pull_request' && 'changed' || 'all' }}
          severity-level: MEDIUM
          confidence-level: HIGH
          fail-on-findings: true
```

## Inputs

| Name               | Type   | Description                                            | Default Value       | Required |
| ------------------ | ------ | ------------------------------------------------------ | ------------------- | -------- |
| `scan-scope`       | String | Scope of files to scan (all/changed)                   | `changed`           | No       |
| `paths`            | String | Paths to scan when using all scope                     | `.`                 | No       |
| `severity-level`   | String | Minimum severity level to report (LOW/MEDIUM/HIGH)     | `LOW`               | No       |
| `confidence-level` | String | Minimum confidence level to report (LOW/MEDIUM/HIGH)   | `LOW`               | No       |
| `output-format`    | String | Format for scan results (plain/json/sarif)             | `plain`             | No       |
| `fail-on-findings` | String | Whether to fail the action if issues are found         | `true`              | No       |
| `upload-sarif`     | String | Whether to upload SARIF results to GitHub Security tab | `true`              | No       |
| `zizmor-version`   | String | Zizmor version                                         | Updated by Renovate | No       |

## Configuration

If necessary, put zizmor configuration into default location `.github/zizmor.yml` - zizmor will discover and use it.

## Outputs

| Name          | Type   | Description                       |
| ------------- | ------ | --------------------------------- |
| `scan_result` | String | Exit code of the Zizmor scan      |
| `report_path` | String | Path to the generated report file |

**Note**: For private repositories without Advanced Security subscription, SARIF upload is automatically skipped, but reports are still available as workflow artifacts or in workflow logs.

## Required permissions

This composite action requires `security-events: write` to upload SARIF results into Security tab.
