# Collect Library Licenses (composite)

This composite action collects license information for Python library dependencies using [pip-licenses](https://github.com/raimon49/pip-licenses), generating a CSV artifact with all dependency licenses.

## Features

- Automatically installs and configures [uv](https://github.com/astral-sh/uv) package manager
- Syncs library dependencies with configurable extras
- Generates a CSV file containing all dependency licenses
- Uploads the license report as a workflow artifact

## Usage

Example usage to collect licenses for a Python library:

```yaml
name: Collect Library Licenses

on:
  push:
    branches:
      - main
  schedule:
    - cron: "0 2 * * 1" # Weekly on Monday

permissions:
  contents: read

jobs:
  collect-licenses:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - name: Collect library licenses
        uses: ./actions/collect-sbom-library
        with:
          path: ./library
```

## Inputs

| Name            | Type   | Description                                                            | Default Value       | Required |
| --------------- | ------ | ---------------------------------------------------------------------- | ------------------- | -------- |
| `path`          | String | Path to the library directory                                          | -                   | Yes      |
| `extras`        | String | Comma-separated list of extras to install (e.g., `full` or `dev,test`) | -                   | No       |
| `artifact-name` | String | Name of the uploaded artifact                                          | `library-licenses`  | No       |
| `uv-version`    | String | Version of uv to install                                               | `0.7.13`            | No       |

## Outputs

This action does not produce outputs, but uploads the following artifact:

| Artifact Name       | Description                                     | Retention |
| ------------------- | ----------------------------------------------- | --------- |
| `library-licenses`  | CSV file containing library dependency licenses | 30 days   |

## Required permissions

No write permissions required.
