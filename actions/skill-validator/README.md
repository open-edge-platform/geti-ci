# Skill Validator (composite)

This composite action validates AI agent skills using [skill-validator](https://github.com/agent-ecosystem/skill-validator), a CLI tool that checks [Agent Skill](https://agentskills.io/) packages for spec compliance, content quality, and link validity.

## Features

- Configurable scan scope: changed skills only (PR), all skills, or automatic detection
- Configurable check groups: structure, links, content, and contamination analysis
- Strict mode: treat warnings as errors for a binary pass/fail gate
- Appends a clean Markdown report to the GitHub Actions job summary (workflow annotation commands are filtered out)
- Emits `::error::` / `::warning::` annotations in real-time per skill for inline PR diff view
- Uploads the Markdown report as a workflow artifact

## What it checks

| Check group     | What it validates |
| --------------- | ----------------- |
| `structure`     | Directory layout, frontmatter fields (`name`, `description`), token limits, code fences, internal links, orphan files |
| `links`         | External HTTP/HTTPS links resolve (HEAD request with 10s timeout) |
| `content`       | Content quality metrics: word count, code block ratio, imperative ratio, instruction specificity |
| `contamination` | Cross-language contamination: code examples that could confuse generation in another language context |

## Usage

> [!IMPORTANT]
> This is a monorepo containing several Actions. When we release the Skill Validator action, we create a tag `skill-validator/v<version>`, e.g. `skill-validator/v0.1.0`.

Example usage that validates changed skills on PRs and all skills on push/schedule:

```yaml
name: Validate Skills

on:
  pull_request:
  push:
    branches:
      - main
  schedule:
    - cron: "0 2 * * *"

permissions: {}

jobs:
  skill-validator:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          persist-credentials: false
          fetch-depth: 0
      - name: Validate skills
        uses: open-edge-platform/geti-ci/actions/skill-validator@<SHA> # skill-validator/v0.1.0
        with:
          skills-path: .github/skills # set to your repo's skills root
          scan-scope: ${{ github.event_name == 'pull_request' && 'changed' || 'all' }}
          strict: ${{ github.event_name == 'pull_request' && 'true' || 'false' }}
          fail-on-findings: ${{ github.event_name == 'pull_request' && 'true' || 'false' }}
```

## Inputs

| Name                | Type   | Description | Default |
| ------------------- | ------ | ----------- | ------- |
| `skills-path`       | String | Repo-relative path to the skills root directory. **Must match where your skills actually live** (e.g. `.github/skills`). If the directory does not exist the action exits successfully with "No skills to validate." | `.agents/skills` |
| `scan-scope`        | String | Scope of skills to validate: `all` (find all), `changed` (git diff only), `auto` (PR→changed, push→all) | `auto` |
| `checks`            | String | Comma-separated list of check groups to run: `structure`, `links`, `content`, `contamination`. Runs all checks when empty. | `` (all) |
| `strict`            | String | Treat warnings as errors (exit 1 instead of 2). Must be exactly `true` or `false`. | `false` |
| `fail-on-findings`  | String | Whether to fail the workflow when validation errors are found. Must be exactly `true` or `false`. | `true` |
| `fail-on-no-skills` | String | Whether to fail if no skills are found at `skills-path`. Catches misconfigured paths. Set to `false` only for repos that genuinely have no skills. Must be exactly `true` or `false`. | `true` |
| `emit-annotations`  | String | Emit GitHub Actions `::error::` / `::warning::` annotations for findings in real-time per skill, surfacing issues inline in the PR diff view. Annotations are separate from the Markdown report so the job summary stays clean. Must be exactly `true` or `false`. | `true` |
| `version`           | String | skill-validator version to install (e.g. `v1.5.6`) | `v1.5.6` |

## Outputs

| Name          | Type   | Description |
| ------------- | ------ | ----------- |
| `exit_code`   | String | `0` — clean pass, or warnings-only with `fail-on-findings: false`; `1` — errors found with `fail-on-findings: true`, or no skills with `fail-on-no-skills: true` |
| `has_errors`  | String | `true` if any validation errors were found, `false` otherwise — independent of `fail-on-findings` |
| `report_path` | String | Path to the generated Markdown report |

## Exit codes

The `skill-validator` binary uses these exit codes:

| Binary exit code | Meaning |
| ---------------- | ------- |
| `0` | Clean pass — no errors, no warnings |
| `1` | Validation errors present (also used for warnings when `strict: true`) |
| `2` | Warnings present, no errors (only when `strict: false`) |

The action accumulates results across all validated skills and maps them to the `exit_code` output as follows:

| Condition | `exit_code` output | Step result |
| --------- | ------------------ | ----------- |
| No errors or warnings | `0` | Success |
| Warnings only (`strict: false`) | `0` | Success |
| Errors found, `fail-on-findings: true` | `1` | Failure |
| No skills found, `fail-on-no-skills: true` | `1` | Failure |
| Errors found, `fail-on-findings: false` | `0` | Success |

> [!NOTE]
> The `has_errors` output always reflects whether errors were found, regardless of `fail-on-findings`. Use it for downstream conditional logic when `fail-on-findings: false`.

## Skills directory layout

Set `skills-path` to the repo-relative path where your skills live. The default (`.agents/skills`) is a convention — override it to match your repository layout:

```yaml
with:
  skills-path: .github/skills # or .claude/skills, skills/, etc.
```

Skills must follow this directory structure under `skills-path`:

```
<skills-path>/
  my-skill/
    SKILL.md   ← required; presence determines a valid skill directory
    ...
```

Each immediate subdirectory containing a `SKILL.md` file is treated as one skill to validate.

## Changed-scope scans (PR mode)

When `scan-scope` is `changed` or `auto` **and the event is a pull request**, the action:

1. Explicitly fetches `origin/<base-ref>` and diffs against the merge base (`origin/<base-ref>...HEAD`) so only changes introduced by the PR are considered — not any commits that landed on the base branch after the PR was created.
2. Only skills that both have a `SKILL.md` **and** contain changed files are validated.

This requires `fetch-depth: 0` on the `actions/checkout` step (shown in the example above).

> [!IMPORTANT]
> Changed-scope diff is **only supported for `pull_request` events**. On `push` and `schedule` events the full skills tree is always scanned, regardless of `scan-scope`. If `scan-scope: changed` is set on a non-PR event, a `::notice::` annotation is emitted and the action falls back to `all`.

> [!NOTE]
> A PR that doesn't touch any skills produces an empty validation list and exits successfully — this is expected, not a misconfiguration. `fail-on-no-skills` does **not** trigger in this case. It only triggers when the `skills-path` directory itself is missing or (on full scans) contains no `SKILL.md` files at all.

## Error handling

| Situation | Behavior |
| --------- | -------- |
| `skills-path` directory does not exist | Emits a `::warning::` annotation; exits `1` if `fail-on-no-skills: true` (default) |
| Directory exists but contains no `SKILL.md` files (full scan) | Emits a `::warning::` annotation; exits `1` if `fail-on-no-skills: true` (default) |
| PR scan where no skills changed | Exits `0` — expected; `fail-on-no-skills` does not apply |
| Base-ref fetch fails (changed scope) | Exits `1` immediately with an error message |
| `fail-on-findings` is not `true` or `false` | Exits `1` immediately with a validation error |
| `fail-on-no-skills` is not `true` or `false` | Exits `1` immediately with a validation error |
| `strict` is not `true` or `false` | Exits `1` immediately with a validation error |
| `emit-annotations` is not `true` or `false` | Exits `1` immediately with a validation error |
| `skills-path` resolves outside `GITHUB_WORKSPACE` | Exits `1` immediately — path traversal rejected |

## Renovate configuration

To keep the action reference pinned to the latest release, add the following to your Renovate config:

```json5
// Custom semver versioning for geti-ci actions to support tags like skill-validator/v0.1.0
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
