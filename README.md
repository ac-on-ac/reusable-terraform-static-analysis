# Reusable Terraform Static Analysis

A reusable GitHub Actions workflow that runs static analysis against Terraform code using `terraform fmt`, [tflint](https://github.com/terraform-linters/tflint), and [checkov](https://www.checkov.io/).

## Purpose

This workflow runs three complementary static analysis tools against a Terraform codebase:

- **`terraform fmt -check`** — verifies all `.tf` files are correctly formatted
- **tflint** — lints Terraform code for provider-specific issues, deprecated syntax, and best-practice violations
- **checkov** — scans for security misconfigurations and compliance issues

`terraform fmt` runs first and must pass before `tflint` and `checkov` proceed. `tflint` and `checkov` run in parallel to minimise total execution time.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `working_directory` | No | `.` | Directory to run all analysis tools against |
| `terraform_version` | No | `latest` | Version of Terraform to install (e.g. `1.11.0`) |
| `tflint_version` | No | `v0.61.0` | Version of tflint to install |
| `checkov_version` | No | `3.2.0` | Version of checkov to install |

## Permissions

The calling workflow must grant these permissions:

| Permission | Reason |
|---|---|
| `contents: read` | Check out the repository |

## Calling this workflow

Minimal — all defaults:

```yaml
name: Terraform Static Analysis

on:
  pull_request:
    types:
      - opened
      - synchronize
      - reopened

permissions:
  contents: read

jobs:
  analysis:
    uses: ac-on-ac/reusable-terraform-static-analysis/.github/workflows/analysis.yml@v1.0.0
```

With explicit versions:

```yaml
jobs:
  analysis:
    uses: ac-on-ac/reusable-terraform-static-analysis/.github/workflows/analysis.yml@v1.0.0
    with:
      working_directory: terraform
      terraform_version: 1.11.0
      tflint_version: v0.61.0
      checkov_version: 3.2.0
```

## Job structure

```
terraform-format
├── tflint   (runs in parallel)
└── checkov  (runs in parallel)
```

`tflint` and `checkov` only run if `terraform-format` passes. They are independent of each other and run concurrently.

## tflint configuration

tflint expects a `.tflint.hcl` configuration file in the `working_directory`. This file controls which plugins are enabled and their settings. Plugins are downloaded during `tflint --init` and cached between runs, keyed on the tflint version and the `.tflint.hcl` file hash.

Example minimal `.tflint.hcl`:

```hcl
plugin "azurerm" {
  enabled = true
  version = "0.28.0"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}
```

tflint is run against every subdirectory containing `.tf` files, excluding `tests/`, `.terraform/`, and `examples/`. If any subdirectory fails, the step fails after all directories have been checked.

## Binary verification

The `tflint` zip is verified against the official `checksums.txt` file published alongside each release before extraction. If the checksum does not match, the workflow fails immediately.

`checkov` is installed via `pip3` at a pinned version to ensure reproducible results.

## Input validation

`working_directory` is validated against the repository root using `realpath` before any tools run. Path traversal values (e.g. `../../other-repo`) are rejected with a clear error.

## Releases

This repository uses the [reusable-manual-release](https://github.com/ac-on-ac/reusable-manual-release) workflow to create releases. Releases are triggered manually from the **Actions** tab.
