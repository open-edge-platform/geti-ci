# Zizmor (composite)

This composite action implements scanning using [Zizmor](https://github.com/zizmorcore/zizmor), providing configurable security analysis with SARIF reporting capabilities.

## Features

- Scans GitHub Actions workflow files for security issues
- Generates both SARIF reports (for GitHub Security tab) and plain/json reports
- Uploads SARIF to Code Scanning for public repositories
- Uploads all reports as workflow artifacts
- Configurable severity/confidence thresholds and failure behavior

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

### Zizmor-specific mappings

Zizmor supports:

- severity levels: `informational`, `low`, `medium`, `high`
- confidence levels: `low`, `medium`, `high`

The following tables summarize minimal severity levels mapping approach for Zizmor.

| Action input severity → | Zizmor severity → | Zizmor SARIF level → | Action security-severity |
| ----------------------- | ----------------- | -------------------- | ------------------------ |
| `LOW`                   | `informational`   | `note`               | `low`                    |
| `MEDIUM`                | `medium`          | `warning`            | `medium`                 |
| `HIGH`                  | `high`            | `error`              | `high`                   |
| `CRITICAL`              | `high`            | `error`              | `high`                   |

For Zizmor severity to SARIF level mapping see Zizmor [code](https://github.com/zizmorcore/zizmor/blob/main/crates/zizmor/src/output/sarif.rs).

The following tables summarize minimal confidence level mapping approach for Zizmor (1:1 mapping).

| Action input confidence → | Zizmor confidence |
| ------------------------- | ----------------- |
| `LOW`                     | `low`             |
| `MEDIUM`                  | `medium`          |
| `HIGH`                    | `high`            |

## Workflow Context Requirements

For proper SARIF upload and PR commenting, this action **MUST** be used within the same workflow and job where GitHub Advanced Security expects results. GitHub Code Scanning correlates SARIF uploads with workflow/job context to provide PR comments and security alerts. Using different workflows or jobs will break this correlation.

## Usage

> [!IMPORTANT] This is a monorepo containing several Actions. When we release Zizmor action, we create a tag `zizmor/v<version>`, e.g. `zizmor/v0.1.0`.

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

permissions: {} # No permissions by default

jobs:
  zizmor-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write # Needed to upload the results to code-scanning dashboard
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          persist-credentials: false
      - name: Run Zizmor scan
        uses: open-edge-platform/geti-ci/actions/zizmor@de9b40c01... # zizmor/v0.1.0
        with:
          scan-scope: ${{ github.event_name == 'pull_request' && 'changed' || 'all' }}
          severity-level: ${{ github.event_name == 'pull_request' && 'HIGH' || 'LOW' }}
          fail-on-findings: ${{ github.event_name == 'pull_request' && 'true' || 'false' }}
```

## Renovate configuration

While Dependabot can update references to these actions, Renovate can't do it out of the box but it can be configured with the following settings (key point is `versioning` flag):

```json
{
  "enabled": true,
  "separateMajorMinor": false,
  "groupName": "GitHub Actions",
  "matchManagers": ["github-actions"],
  "matchPackagePatterns": ["*"],
  "versioning": "regex:^(?:[^/]+/)?v?(?<major>\\d+)\\.(?<minor>\\d+)\\.(?<patch>\\d+)$",
  "schedule": ["* * 1 * *"] // every month
}
```

## Inputs

| Name               | Type   | Description                                                 | Default Value       | Required |
| ------------------ | ------ | ----------------------------------------------------------- | ------------------- | -------- |
| `scan-scope`       | String | Scope of files to scan (all/changed)                        | `changed`           | No       |
| `paths`            | String | Paths to scan when using all scope                          | `.`                 | No       |
| `severity-level`   | String | Minimum severity level to report (LOW/MEDIUM/HIGH/CRITICAL) | `LOW`               | No       |
| `confidence-level` | String | Minimum confidence level to report (LOW/MEDIUM/HIGH)        | `LOW`               | No       |
| `output-format`    | String | Format for scan results (plain/json/sarif)                  | `plain`             | No       |
| `fail-on-findings` | String | Whether to fail the action if issues are found              | `true`              | No       |
| `upload-sarif`     | String | Whether to upload SARIF results to GitHub Security tab      | `true`              | No       |
| `zizmor-version`   | String | Zizmor version                                              | Updated by Renovate | No       |

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
