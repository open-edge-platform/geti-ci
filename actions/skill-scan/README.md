# Skill Security Scan (composite)

This composite action scans AI agent skills using [SkillSpector](https://github.com/NVIDIA/SkillSpector), a static analysis tool that detects security issues in skill definitions.

## Features

- Configurable scan scope: changed skills only (PR), all skills, or automatic detection
- Configurable severity threshold and failure behavior
- Uploads a plain-text report as a workflow artifact

## Usage

> [!IMPORTANT] This is a monorepo containing several Actions. When we release the Skill Security Scan action, we create a tag `skill-scan/v<version>`, e.g. `skill-scan/v0.1.0`.

Example usage that scans changed skills on PRs and all skills on push/schedule:

```yaml
name: Skill Security Scan

on:
  pull_request:
  push:
    branches:
      - main
  schedule:
    - cron: "0 2 * * *"

permissions: {}

jobs:
  skill-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          persist-credentials: false
          fetch-depth: 0
      - name: Run Skill Security Scan
        uses: open-edge-platform/geti-ci/actions/skill-scan@<SHA> # skill-scan/v0.1.0
        with:
          scan-scope: ${{ github.event_name == 'pull_request' && 'changed' || 'all' }}
          severity-level: ${{ github.event_name == 'pull_request' && 'HIGH' || 'LOW' }}
          fail-on-findings: ${{ github.event_name == 'pull_request' && 'true' || 'false' }}
```

## Inputs

| Name               | Type   | Description                                                                                     | Default                                      | Required |
| ------------------ | ------ | ----------------------------------------------------------------------------------------------- | -------------------------------------------- | -------- |
| `severity-level`   | String | Minimum skill severity level to fail on (LOW/MEDIUM/HIGH/CRITICAL)                              | `CRITICAL`                                   | No       |
| `fail-on-findings` | String | Whether to fail the workflow when findings meet the threshold                                   | `true`                                       | No       |
| `skills-path`      | String | Repo-relative path to the skills root directory                                                 | `.agents/skills`                             | No       |
| `scan-scope`       | String | Scope of skills to scan: `all` (find all), `changed` (git diff only), `auto` (PR→changed, push→all) | `auto`                                  | No       |
| `version`          | String | SkillSpector pinned commit SHA to install                                                       | Updated manually (no releases available yet) | No       |

## Outputs

| Name          | Type   | Description                         |
| ------------- | ------ | ----------------------------------- |
| `exit_code`   | String | Exit code of the scan step (0 or 1) |
| `report_path` | String | Path to the generated scan report   |

## Severity levels

The `severity-level` input controls which overall skill risk ratings trigger a failure:

| `severity-level` | Fails when skill severity is |
| ---------------- | ---------------------------- |
| `LOW`            | LOW, MEDIUM, HIGH, CRITICAL  |
| `MEDIUM`         | MEDIUM, HIGH, CRITICAL       |
| `HIGH`           | HIGH, CRITICAL               |
| `CRITICAL`       | CRITICAL only                |

The threshold is applied to the **overall skill severity** returned by SkillSpector, not to individual issue severities within a skill.

## Skills directory layout

Skills must follow this directory structure under `skills-path`:

```
.agents/skills/
  my-skill/
    SKILL.md      ← required; presence determines a valid skill directory
    ...
```

Each immediate subdirectory containing a `SKILL.md` file is treated as one skill to scan.

## Renovate configuration

To keep the action reference pinned to the latest release, add the following to your Renovate config:

```json5
// Custom semver versioning for geti-ci actions to support tags like skill-scan/v0.1.0
{
  groupName: "GitHub Actions",
  matchManagers: ["github-actions"],
  matchPackageNames: ["open-edge-platform/geti-ci"],
  schedule: ["* * 1 * *"], // every month
  versioning: "regex:^(?<compatibility>[^/]+)/v?(?<major>\\d+)\\.(?<minor>\\d+)\\.(?<patch>\\d+)$",
},
```

## Required permissions

This action requires only `contents: read`.
