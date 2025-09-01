# Pre-commit Quality Action (composite)

This composite action executes pre-commit hooks for code quality checks with configurable Python and Node.js environments.

## Usage

```yaml
- name: Checkout code
  uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2
  with:
    persist-credentials: false
- name: Run pre-commit check
  uses: ./actions/pre-commit
```

## Inputs

| Name             | Type   | Description                           | Default Value | Required |
| ---------------- | ------ | ------------------------------------- | ------------- | -------- |
| `python-version` | String | Python version to use                 | `3.11`        | No       |
| `node-version`   | String | Node.js version to use                | `20`          | No       |
| `skip`           | String | Comma-separated list of hooks to skip | ``            | No       |
| `cache`          | String | Whether to use caching                | `true`        | No       |

## Outputs

| Name        | Type   | Description               |
| ----------- | ------ | ------------------------- |
| `cache-hit` | String | Whether the cache was hit |

## Required permissions

No write permissions required.
