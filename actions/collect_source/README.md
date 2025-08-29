# Collect source code (composite)

This composite action executes GitHub Actions workflows gathering source code of packages with GPL or MPL license.

## Usage

Example usage in a repository on merge into main and release branches:

```yaml
name: Collect source code

on:
  push:
    branches:
      - main
      - release**

jobs:
  collect source code:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Collect source code
        uses: ./actions/collect_source
        with:
          images: "ghcr.io/open-edge-platform/geti/account-service:2.12.1,"ghcr.io/open-edge-platform/geti/credit-service:2.12.1","ghcr.io/open-edge-platform/geti/director:2.12.1""
          archive-name: source_archive.tgz
          
```

## Inputs

| Name               | Type    | Description                                          | Default Value       | Required |
| ------------------ | ------- | ---------------------------------------------------- | ------------------- | -------- |
| `images`           | String  | Comma-separated list of container images to inspect  |                     | Yes      |
| `archive-name`     | String  | File name to store the archived source code          |                     | Yes      |
| `output-dir`       | String  | Path to store temporarily the downloaded source code | `output`            | No       |
