# Cleanup Runner Resources (composite)

This composite action frees up disk space and cleans Docker resources on a GitHub Actions runner.

## Usage

```yaml
- name: Run pre-commit check
  uses: ./actions/pre-commit
```

## Inputs

| Name   | Type   | Description                                      | Default Value | Required |
| ------ | ------ | ------------------------------------------------ | ------------- | -------- |
| `type` | String | Type of cleanup to perform: initial or pre-build | `initial`     | True     |

## Outputs

No outputs.

## Required permissions

No write permissions required.
