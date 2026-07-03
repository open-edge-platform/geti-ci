# Skill Security Scan (composite)

This composite action scans AI agent skills using [SkillSpector](https://github.com/NVIDIA/SkillSpector), a static analysis tool that detects security issues in skill definitions.

## Features

- Configurable scan scope: changed skills only (PR), all skills, or automatic detection
- Configurable severity threshold and failure behavior
- Uploads a plain-text report as a workflow artifact

## Usage

> [!IMPORTANT]
> This is a monorepo containing several Actions. When we release the Skill Security Scan action, we create a tag `skill-scan/v<version>`, e.g. `skill-scan/v0.1.0`.

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
          skills-path: .github/skills          # set to your repo's skills root
          scan-scope: ${{ github.event_name == 'pull_request' && 'changed' || 'all' }}
          severity-level: ${{ github.event_name == 'pull_request' && 'HIGH' || 'LOW' }}
          fail-on-findings: ${{ github.event_name == 'pull_request' && 'true' || 'false' }}
```

## Inputs

| Name               | Type   | Description                                                                                         | Default                                      | Required |
| ------------------ | ------ | --------------------------------------------------------------------------------------------------- | -------------------------------------------- | -------- |
| `severity-level`   | String | Minimum skill severity level to fail on (LOW/MEDIUM/HIGH/CRITICAL)                                  | `CRITICAL`                                   | No       |
| `fail-on-findings` | String | Whether to fail the workflow when findings meet the threshold. Must be exactly `true` or `false` — any other value causes an immediate error. | `true` | No |
| `skills-path`      | String | Repo-relative path to the skills root directory. **Must match where your skills actually live** (e.g. `.github/skills`). If the directory does not exist the action exits successfully with "No skills to scan." | `.agents/skills` | No |
| `scan-scope`       | String | Scope of skills to scan: `all` (find all), `changed` (git diff only), `auto` (PR→changed, push→all) | `auto`                                       | No       |
| `fail-on-no-skills` | String | Whether to fail if no skills are found at `skills-path`. Catches misconfigured paths. Set to `false` only for repos that genuinely have no skills. Must be exactly `true` or `false`. | `true` | No |
| `version`          | String | SkillSpector pinned commit SHA to install                                                           | Updated manually (no releases available yet) | No       |

## Outputs

| Name                | Type   | Description                                                                                         |
| ------------------- | ------ | --------------------------------------------------------------------------------------------------- |
| `exit_code`         | String | Exit code of the scan step: `0` (no action needed) or `1` (findings at or above threshold, and `fail-on-findings: true`) |
| `findings_exceeded` | String | `true` if any skill's severity met or exceeded the threshold **or** if SkillSpector output could not be parsed, `false` otherwise — independent of `fail-on-findings` |
| `report_path`       | String | Path to the generated scan report                                                                   |

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

Set `skills-path` to the repo-relative path where your skills live. The default (`.agents/skills`) is just a convention — override it to match your repository layout:

```yaml
with:
  skills-path: .github/skills   # or .agents/skills, skills/, etc.
```

Skills must follow this directory structure under `skills-path`:

```
<skills-path>/
  my-skill/
    SKILL.md      ← required; presence determines a valid skill directory
    ...
```

Each immediate subdirectory containing a `SKILL.md` file is treated as one skill to scan.

If `skills-path` does not exist in the repository the action exits successfully and reports "No skills to scan." rather than failing.

## Changed-scope scans (PR mode)

When `scan-scope` is `changed` (or `auto` on a pull request), the action:

1. Explicitly fetches `origin/<base-ref>` before computing the diff. The step **fails immediately** if the fetch fails, rather than silently producing an empty skill list.
2. Diffs `origin/<base-ref>...HEAD` using a repo-relative pathspec to find skills that changed in the PR.

This requires `fetch-depth: 0` on the `actions/checkout` step (shown in the example above).

## Error handling

| Situation | Behavior |
| --------- | -------- |
| `skills-path` directory does not exist | Emits a `::warning::` annotation; exits `1` if `fail-on-no-skills: true` (default) |
| Directory exists but contains no `SKILL.md` files | Emits a `::warning::` annotation; exits `1` if `fail-on-no-skills: true` (default) |
| Base-ref fetch fails (changed scope) | Exits `1` immediately with an error message |
| SkillSpector output is not valid JSON | Sets `findings_exceeded=true`; continues scanning remaining skills |
| `fail-on-findings` is not `true` or `false` | Exits `1` immediately with a validation error |
| `fail-on-no-skills` is not `true` or `false` | Exits `1` immediately with a validation error |
| `severity-level` is not a valid level | Exits `1` immediately with a validation error |

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
